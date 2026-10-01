# Lesson 14 — Routing and Controllers

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Given any URL, find the Rails **route**, **controller**, and **action** that handle it, and read what that action does (params, `before_action`, `render json:`, status codes).
**You need:** Lessons 1–13 (especially Ruby classes/modules, blocks, and ActiveRecord). Open `~/Josh_Files/freee-payroll/` in your editor.

---

## 1. The big idea: from URL to code

In Lesson 9 you saw the life of a request:

```
Browser ──HTTP request──▶ Router (config/routes.rb) ──▶ Controller#action ──▶ Models / Services / Repositories
   ▲                                                          │
   └──────────────────── HTTP response (JSON or HTML) ◀───────┘
```

This lesson zooms in on the first two boxes: the **router** and the **controller**.

| Term | Meaning in plain words | Example in freee-payroll |
|---|---|---|
| **Route** | One rule: "this HTTP method + this URL path goes to this controller action." | `GET /api/p/employee_display_names/:id` → `Api::Private::EmployeeDisplayNamesController#show` |
| **Router / routing** | The part of Rails that reads all the routes and picks the matching one for each request. | `config/routes.rb` + `config/routes/*.rb` |
| **Controller** | A Ruby class that receives a request and builds a response. Name ends in `Controller`. | `app/controllers/api/private/comments_controller.rb` |
| **Action** | A public method in a controller that handles one kind of request. | `def show`, `def create` |
| **`controller#action`** | Short way to write "this action in this controller". | `web/holidays#index` |
| **Path parameter** | A variable piece of the URL path, written `:name` in a route. | `:id` in `/api/p/comments/:id` |
| **`params`** | A hash-like object holding everything the client sent: path params, query string, and body. | `params[:id]`, `params[:url]` |
| **DSL** | "Domain-Specific Language": Ruby code that reads like a mini-language made for one job. `routes.rb` is a routing DSL. | `resources :comments, only: %i[index create destroy]` |

---

## 2. Where routes live in freee-payroll

The main file is `config/routes.rb`. It is **huge** (about 3,600 lines), so it is split with `draw`:

```ruby
# config/routes.rb:1-8
Rails.application.routes.draw do
  # For details on the DSL available within this file, see https://guides.rubyonrails.org/routing.html

  draw :web
  draw :admin
  draw :hr_master

  namespace :api, defaults: { format: :json } do
```

- `draw :web` means "also load `config/routes/web.rb`". The split files are in `config/routes/`: `web.rb`, `admin.rb`, `hr_master.rb`, `public_api_v0.rb`, `public_api_v1.rb`, `internal_api.rb`, `mobile_api.rb`, `admin_api.rb`, `debug.rb`.
- `defaults: { format: :json }` means everything inside `/api` is treated as a JSON request by default.

### 2.1 `namespace` and `scope`: building the URL prefix

| DSL word | Adds to the URL | Adds to the controller's module/folder | Example |
|---|---|---|---|
| `namespace :api` | `/api` | `Api::` → `app/controllers/api/` | `namespace :api do ... end` |
| `scope 'p'` | `/p` | nothing | — |
| `scope 'p', module: :private` | `/p` | `Private::` → `.../private/` | used for the Private API |
| `scope ':year/:month'` | `/:year/:month` (path params) | nothing | `config/routes.rb:899` |

So the Private API is built like this:

```ruby
# config/routes.rb:8 and :34
namespace :api, defaults: { format: :json } do   # URL /api   → module Api
  ...
  scope 'p', module: :private do                 # URL /api/p → module Api::Private
    ...
    resources :comments, only: %i[index create destroy]   # config/routes.rb:234
    ...
    resources :employee_display_names, only: [:show]      # config/routes.rb:236
    ...
  end # api private end                                   # config/routes.rb:3566
```

Result: the URL `/api/p/comments` goes to `Api::Private::CommentsController`, found in `app/controllers/api/private/comments_controller.rb`.

> 🔑 **The naming rule (Rails convention):** path segment `api/private/comments` ⇄ class `Api::Private::CommentsController` ⇄ file `app/controllers/api/private/comments_controller.rb`. If you know one, you can work out the other two.

---

## 3. `resources`: the REST shortcut

**REST** (REpresentational State Transfer) is a common convention: every "thing" (a **resource**) gets the same standard set of URLs. `resources :comments` creates up to **7 routes** at once:

| HTTP method | Path | Action | Meaning |
|---|---|---|---|
| GET | `/comments` | `index` | List all |
| GET | `/comments/:id` | `show` | Show one |
| GET | `/comments/new` | `new` | HTML form for a new one (rare in APIs) |
| POST | `/comments` | `create` | Create one |
| GET | `/comments/:id/edit` | `edit` | HTML edit form (rare in APIs) |
| PATCH / PUT | `/comments/:id` | `update` | Update one |
| DELETE | `/comments/:id` | `destroy` | Delete one |

freee-payroll almost always limits them with `only:`:

```ruby
# config/routes.rb:234
resources :comments, only: %i[index create destroy]
```

This makes only three routes: `GET /api/p/comments` → `index`, `POST /api/p/comments` → `create`, `DELETE /api/p/comments/:id` → `destroy`.

> `%i[index create destroy]` is Ruby shorthand for `[:index, :create, :destroy]` (an array of symbols, from Lesson 4).

### 3.1 `resource` (singular): when there is only one

Some things exist **once per company**, like a company setting. For those there's no `:id` in the URL:

```ruby
# config/routes.rb:424
resource :company_daily_prorate_setting, only: %i[show update]
```

| Method | Path | Action |
|---|---|---|
| GET | `/api/p/company_daily_prorate_setting` | `show` |
| PATCH/PUT | `/api/p/company_daily_prorate_setting` | `update` |

Note: the controller name is still **plural**: `CompanyDailyProrateSettingsController`. That's a Rails rule for `resource`.

### 3.2 `member` and `collection`: extra, non-standard actions

- **collection** route = acts on the whole list, **no** `:id` in the path.
- **member** route = acts on one item, **has** `:id` in the path.

```ruby
# config/routes.rb:257-268
namespace :employee_shifts do
  resources :imports, only: [:index] do
    collection do
      get :importing_status
      get :template
      post :prepare
    end
    member do
      get :download
      put :execute_async
    end
  end
```

This creates, for example:

| Method | Path | Controller#action |
|---|---|---|
| GET | `/api/p/employee_shifts/imports` | `api/private/employee_shifts/imports#index` |
| GET | `/api/p/employee_shifts/imports/importing_status` | `...#importing_status` (collection) |
| POST | `/api/p/employee_shifts/imports/prepare` | `...#prepare` (collection) |
| GET | `/api/p/employee_shifts/imports/:id/download` | `...#download` (member) |
| PUT | `/api/p/employee_shifts/imports/:id/execute_async` | `...#execute_async` (member) |

### 3.3 Plain `get` / `post` with `to:`

When REST doesn't fit, a route can be written by hand. `to: 'controller#action'`:

```ruby
# config/routes.rb:895
get 'attendance_support_banner', to: 'attendance_support_banner#show'
```
→ `GET /api/p/attendance_support_banner` → `Api::Private::AttendanceSupportBannerController#show`.

### 3.4 `param:` and `constraints:`

```ruby
# config/routes.rb:921-923
resources :payroll_statements, only: %i[index show update], param: :emp_id do
  resource :remark, controller: 'payroll_statements/remarks', only: [:update]
end
```
- `param: :emp_id` renames the path parameter: `/payroll_statements/:emp_id` instead of `:id`. In the controller you read `params[:emp_id]`.
- `controller:` says which controller to use when it's not the default name.
- Routes can be **nested**: a `resource` inside `resources` gives `/payroll_statements/:payroll_statement_emp_id/remark`.

```ruby
# config/routes.rb:899-907
scope ':year/:month' do
  resource :license_target_employee_count,
           controller: 'employees/license_target_employee_counts',
           only: [:show],
           constraints: {
             year: /\d{4}/,
             month: /(?:[1-9]|1[0-2])/,
           }
end
```
- `constraints:` uses **regular expressions** (patterns for text, written between `/ /`). `year: /\d{4}/` means "exactly 4 digits". A URL like `/api/p/abc/13/license_target_employee_count` will **not** match.

### 3.5 Web page routes and "catch-all" paths

HTML pages live in `config/routes/web.rb`. Most are one flat line each:

```ruby
# config/routes/web.rb:1
root to: 'web/homes#index'
# config/routes/web.rb:39
get 'employees', to: 'web/employees#index'
# config/routes/web.rb:44
get 'attendances(/*path)', to: 'web/attendances#index'
# config/routes/web.rb:46
get 'holidays', to: 'web/holidays#index'
```

- `root` = the page for `/`.
- `(/*path)` = "optionally, **anything** after this". `/attendances`, `/attendances/2026/09`, and `/attendances/a/b/c` all go to `web/attendances#index`. **Why?** The page is a React app with its **own** router in the browser (react-router, Lesson 30). Rails only has to serve the same empty page, and React then decides what to show based on the rest of the URL.

---

## 4. Finding a route: tools

| Way | Command / action | When |
|---|---|---|
| List all routes | `bin/rails routes` | Big list (thousands). Slow. |
| Filter by text | `bin/rails routes -g display_names` | `-g` = grep. Shows only matching lines. |
| Filter by controller | `bin/rails routes -c api/private/comments` | See every URL one controller handles. |
| Search the file | search `resources :comments` in `config/routes.rb` | Fast, no server needed. |
| Claude skill | the repo has `.claude/skills/find-route/` | Give it a path like `GET /api/p/companies`. It finds the controller, action, and swagger file. |

Output of `bin/rails routes -g employee_display_names` looks like this (illustrative example):

```
                    Prefix Verb URI Pattern                                  Controller#Action
 api_employee_display_name GET  /api/p/employee_display_names/:id(.:format)  api/private/employee_display_names#show {:format=>:json}
```

Read it as: **GET** + that **URI pattern** → controller `api/private/employee_display_names`, action `show`.

---

## 5. Controllers

### 5.1 The inheritance chain

Every controller **inherits** (Lesson 6, `<`) from a parent that adds shared behavior:

```
ActionController::Base          (Rails)
  └─ ApplicationController      app/controllers/application_controller.rb  (login, auth, errors)
       ├─ Api::BaseController   app/controllers/api/base_controller.rb
       │    └─ Api::Private::BaseController   app/controllers/api/private/base_controller.rb
       │         └─ Api::Private::CommentsController   (a "leaf" controller: the real endpoint)
       └─ Web::BaseController   app/controllers/web/base_controller.rb
            └─ Web::HolidaysController
```

| Term | Meaning |
|---|---|
| **Base controller** | A parent class holding code shared by many controllers. You rarely write one. |
| **Leaf controller** | The concrete controller at the bottom of the tree that actually handles an endpoint. |
| **Concern** (controller) | A module mixed in with `include` to share behavior (Lesson 13), e.g. `ErrorHandleable`. |

### 5.2 A tiny, complete controller

```ruby
# app/controllers/api/private/attendance_support_banner_controller.rb
# typed: true
class Api::Private::AttendanceSupportBannerController < Api::Private::BaseController
  include ErrorHandleable

  def show
    unless Omega::FeatureFlag.active?(:attendance_navigation_v1, namespace: :payroll)
      head :not_implemented
      return
    end

    render json: { show_attendance_support_banner: show_attendance_support_banner? }
  end

  private

  def show_attendance_support_banner?
    # TODO: 勤怠サポートバナーの表示可否を判定するロジックを実装する
    false
  end
end
```

Line by line:
- `< Api::Private::BaseController`: it's a Private API controller.
- `include ErrorHandleable`: standard error handling (Lesson 15). **Required**: `spec/controller_class_policies_spec.rb` checks that every controller includes `ErrorHandleable` or `Omega::Controller::ErrorHandler` (rule from `app/CLAUDE.md`).
- `def show`: the action. Public methods = actions.
- `head :not_implemented`: respond with only a status (501), no body. Then `return` stops the method.
- `render json: { ... }`: turn a Ruby hash into JSON and send it with status 200 (the default).
- `private`: everything below is a helper, **not** reachable as an action.

### 5.3 Ways to respond

| Code | What the client gets | Real example |
|---|---|---|
| `render json: hash` | JSON body, status 200 | `attendance_support_banner_controller.rb` |
| `render json: res, status: :ok` | Same, status written explicitly | `app/controllers/api/private/comments_controller.rb:23` |
| `head :created` | No body, status 201 | `comments_controller.rb:42` |
| `head :no_content` | No body, status 204 | `comments_controller.rb:58` |
| `render json: {...}, status: :unauthorized` | JSON with status 401 | `app/controllers/api/private/base_controller.rb:29` |
| `render template: 'web/app'` | An HTML page (Lesson 16) | `app/controllers/web/holidays_controller.rb:6` |

Rails lets you write statuses as **symbols**: `:ok` 200, `:created` 201, `:no_content` 204, `:bad_request` 400, `:unauthorized` 401, `:forbidden` 403, `:not_found` 404, `:unprocessable_entity` 422, `:internal_server_error` 500, `:not_implemented` 501.

> ⚠️ An action must respond **once**. Calling `render` twice raises `DoubleRenderError`. That's why you see `render ... and return` or `head ...; return`.

### 5.4 `params` and strong parameters

`params` collects **everything** the client sent:

| Source | Example request | Read with |
|---|---|---|
| Path | `GET /api/p/comments/5` | `params[:id]` → `"5"` (a **String**!) |
| Query string | `GET /api/p/comments?url=/employees` | `params[:url]` |
| JSON body | `POST` with `{"comment":"hi"}` | `params[:comment]` |

> Values come in as strings. That's why you see `params[:id].to_i` (`comments_controller.rb:49`).

**Strong parameters** is a Rails safety feature. Before passing user input to a model, you must **whitelist** which keys are allowed. This prevents **mass assignment attacks**: a user can't slip in `{"company_id": 999}` and change data they shouldn't.

| Method | Meaning |
|---|---|
| `params.require(:x)` | "Key `:x` **must** exist". If it's missing → 400 error. Returns what's inside `:x`. |
| `.permit(:a, :b)` | Keep only keys `a` and `b`. Everything else is dropped. |
| `permit(dates: [])` | Allow `dates` to be an **array** of simple values. |
| `permit(address: %i[zipcode street_name1])` | Allow a **nested hash** with only those keys. |
| `.to_h` | Turn the permitted params into a normal Ruby Hash. |

Real examples:

```ruby
# app/controllers/api/private/company_daily_prorate_settings_controller.rb:13
permitted_params = params.permit(:round_type)
```

```ruby
# app/controllers/api/private/company_prescribed_holidays_controller.rb:236-238
def company_prescribed_holidays_params
  params.permit(:name, dates: [])
end
```

```ruby
# app/controllers/api/private/settings_controller.rb:33-34 and :38-54 (shortened)
company_profile.attributes =
  params.require(:company_profile).permit(:payroll_contact_department, :payroll_contact_name)
...
auth_company_params =
  params
    .require(:auth_company)
    .permit(
      :name,
      :name_kana,
      ...
      address: %i[zipcode street_name1 street_name2 street_name1_kana street_name2_kana prefecture_code],
      phone1: %i[a b c],
    )
    .to_h
    .symbolize_keys
```

Notice the pattern: permit in a **private helper method** named `xxx_params`, or inline as `permitted_params = params.permit(...)`.

### 5.5 `before_action`: code that runs before every action

A **filter** (also called a **callback**) is a method Rails runs automatically around your action.

| Filter | Runs |
|---|---|
| `before_action :m` | Before the action. If it calls `render`/`redirect`, the action is **skipped**. |
| `after_action :m` | After the action. |
| `around_action :m` | Wraps the action (before + after). |
| `prepend_before_action :m` | Before, but put at the **front** of the line. |
| `skip_before_action :m` | Turn off a filter inherited from a parent. |

`ApplicationController` sets up many filters (shortened):

```ruby
# app/controllers/application_controller.rb:46-73
prepend_before_action :set_cache_buster
around_action :record_error_for_activity_log
before_action :set_omega_auth_context
...
before_action :check_authenticated          # line 57
...
before_action :check_authorized             # line 61
...
before_action :has_allowed_ac?              # line 64
...
after_action :set_csrf_cookie
...
rescue_from StandardError, with: :handle_error   # line 73
```

A leaf controller can add its own:

```ruby
# app/controllers/api/private/company_prescribed_holidays_controller.rb (lines 4-7 and 230-234)
class Api::Private::CompanyPrescribedHolidaysController < Api::Private::BaseController
  include ErrorHandleable

  before_action :validate_usable
  ...
  def validate_usable
    unless feature_usable?(:company_prescribed_holidays)
      raise ::HrErrors::UserError.new('アクセス権限がありません。', status: :forbidden)
    end
  end
```

### 5.6 Authentication vs authorization

| Term | Question it answers | Where in freee-payroll |
|---|---|---|
| **Authentication** (認証) | "*Who* are you? Are you logged in?" | `check_authenticated` |
| **Authorization** (認可) | "Are you *allowed* to do this?" | `check_authorized`, `has_allowed_ac?` |

`ApplicationController` **forces** each subclass to decide:

```ruby
# app/controllers/application_controller.rb:358-370
# 認証チェックフィルター
# チェック漏れのリスクを低減するため ApplicationController のフィルタとして呼び出します。
# 各 Controller で実装を override またはチェック不要なら skip してください。
def check_authenticated
  raise NotImplementedError
end

# 認可チェックフィルター
...
def check_authorized
  raise NotImplementedError
end
```

`raise NotImplementedError` = "a child class must **override** this." (Overriding = defining a method with the same name in a subclass, Lesson 6.) The API base does it with a JSON 401:

```ruby
# app/controllers/api/private/base_controller.rb:26-30
# @override
def check_authenticated
  # ログイン状態かどうか
  render json: { message: 'タイムアウトしました。設定よりログアウトのうえで、再度ログインして下さい。' }, status: :unauthorized unless current_auth_user
end
```

…while `Web::BaseController` **redirects** to the login page instead (`app/controllers/web/base_controller.rb`, `check_authenticated`: `redirect_to users_login_path`). APIs answer with status codes; web pages redirect.

#### Endpoint permissions: `config/permission.yaml`

`has_allowed_ac?` (`application_controller.rb:138`) looks up the current `controller` + `action` in `config/permission.yaml` and checks whether the user has one of the required **function keys** (permission names):

```yaml
# config/permission.yaml:1650-1653
  - controller: api/private/employee_display_names
    action: show
    function_keys:
      - employee:contract:write
```

If the user lacks the permission → `HrErrors::UserError.new('アクセス権限がありません。', status: :forbidden)` → **403**.

> 📌 Team rule (`app/controllers/api/private/README.md`): **every new endpoint must be registered in `config/permission.yaml`**. `config/sekisyo.yml` is auto-generated from it, so don't edit it by hand. Also, don't write tests that just copy `permission.yaml` (`spec/controllers/CLAUDE.md`).

### 5.7 `current_company`, `current_auth_user`, `helper_method`

Inside any controller you can call:

| Method | Returns |
|---|---|
| `current_auth_user` | The logged-in user (from the auth service) |
| `current_company` | The `Company` (事業所) the user is working in (an ActiveRecord model) |
| `current_employee` | The `Employee` record of the logged-in user, if any |

```ruby
# app/controllers/application_controller.rb:19-22 (start of the list)
helper_method :current_user,
              :current_auth_user,
              :current_auth_company,
              :current_company,
```

`helper_method` makes these controller methods **also usable in views** (Lesson 16).

> 🔑 **Always scope data by `current_company`.** Controllers pass `company_id: current_company.id` to repositories and services. That keeps one customer from seeing another's data (multi-tenancy, Lesson 1).

---

## 6. Putting it together: one real controller

```ruby
# app/controllers/api/private/company_daily_prorate_settings_controller.rb (full file)
class Api::Private::CompanyDailyProrateSettingsController < Api::Private::BaseController
  include ErrorHandleable

  def show
    company_daily_prorate_setting =
      ::Payroll::Infrastructure::Repositories::CompanyDailyProrateSettingRepository.find_one_by_company_id(
        company_id: current_company.id,
      )
    render json: build_response(company_daily_prorate_setting: company_daily_prorate_setting), status: :ok
  end

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

  private

  def build_response(company_daily_prorate_setting:)
    { round_type: company_daily_prorate_setting.round_type }
  end
end
```

How to read it:
1. **Route:** `resource :company_daily_prorate_setting, only: %i[show update]` (`config/routes.rb:424`) → `GET` and `PUT/PATCH /api/p/company_daily_prorate_setting`.
2. **show:** ask a **Repository** (Lesson 17) for the setting of `current_company`, then return `{ round_type: ... }`.
3. **update:** permit only `round_type`, then call a **Service** (Lesson 17) with `.execute`. The comment explains why access control is turned off: the change triggers payroll recalculation for **all** employees.
4. **private build_response:** one place that shapes the JSON, reused by both actions.

> Notice what the controller does **not** do: no business rules, no SQL. It **receives, delegates, responds**. That's the freee-payroll style: *thin controllers*.

---

## 7. Practice (15 min)

**Practice 1: route → controller.** For each route, write the HTTP method + full URL and the `Controller#action` (full class name).

```ruby
# inside namespace :api ... scope 'p', module: :private
a) resources :comments, only: %i[index create destroy]
b) resource :company_daily_prorate_setting, only: %i[show update]
c) get 'attendance_support_banner', to: 'attendance_support_banner#show'
```

**Practice 2: find it yourself.** In `config/routes.rb`, search `resources :company_prescribed_holidays`. List every URL it creates. (Hint: two standard actions plus a `collection` block, `config/routes.rb:3129-3137`.)

**Practice 3: strong params.** A client sends `{"name": "夏季休暇", "dates": ["2026-08-13", "2026-08-14"], "company_id": 999}`. What does `params.permit(:name, dates: [])` keep?

**Practice 4: write it.** (illustrative example) Write a controller action `show` for `Api::Private::HelloController` that returns JSON `{ "message": "hello", "company_id": <current company id> }`.

<details><summary>Practice answers</summary>

1. a) `GET /api/p/comments` → `Api::Private::CommentsController#index`; `POST /api/p/comments` → `#create`; `DELETE /api/p/comments/:id` → `#destroy`.
   b) `GET /api/p/company_daily_prorate_setting` → `Api::Private::CompanyDailyProrateSettingsController#show`; `PATCH`/`PUT` same URL → `#update`.
   c) `GET /api/p/attendance_support_banner` → `Api::Private::AttendanceSupportBannerController#show`.
2. `GET /api/p/company_prescribed_holidays` (index), `POST /api/p/company_prescribed_holidays` (create), `POST .../validate`, `POST .../reflect`, `PATCH /api/p/company_prescribed_holidays` (update, collection), `DELETE /api/p/company_prescribed_holidays` (destroy, collection), `GET .../upcoming`.
3. `{ "name" => "夏季休暇", "dates" => ["2026-08-13", "2026-08-14"] }`. `company_id` is dropped.
4.
```ruby
class Api::Private::HelloController < Api::Private::BaseController
  include ErrorHandleable

  def show
    render json: { message: 'hello', company_id: current_company.id }
  end
end
```
</details>

---

## 8. Summary

- **Routes** (`config/routes.rb`, split with `draw`) map *method + path* → `controller#action`.
- `namespace :api` + `scope 'p', module: :private` = `/api/p/...` → `Api::Private::...Controller`.
- `resources` = up to 7 REST routes (limit with `only:`); `resource` = singular, no `:id`; `member` has `:id`, `collection` doesn't.
- Controllers inherit from base controllers; **public methods are actions**; helpers go under `private`.
- Respond with `render json:`, `head :status`, or `render template:`, and **only once**.
- **Strong params** (`require` / `permit`) whitelist input.
- `before_action` runs filters; `check_authenticated` (who are you?) and `check_authorized` / `has_allowed_ac?` + `config/permission.yaml` (are you allowed?).
- Every controller must `include ErrorHandleable` (or Omega's handler). Always scope by `current_company`.
- freee-payroll controllers are **thin**: receive → delegate to service/repository → respond.

➡️ Now take **Test14.md**.
