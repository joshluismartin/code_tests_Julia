# Lesson 9 — Rails Overview: MVC and the Request Lifecycle

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Understand what Rails is, how it organizes code (MVC + conventions), how file names turn into class names, and the big-picture path a request takes through freee-payroll.
**You need:** Lessons 1–8 (Ruby + Sorbet). Open `~/Josh_Files/freee-payroll/` in your editor.

---

## 1. What is Rails?

| Term | Meaning in plain words |
|---|---|
| **Ruby on Rails ("Rails")** | A **framework** for building web backends in Ruby. It gives you ready-made pieces: routing, database access, JSON rendering, tests... You fill in your app's specific logic. |
| **Framework** (recap) | Code that *calls your code*. You follow its rules and it handles the boring parts. |
| **Gem** (recap) | A Ruby package. Rails itself is a gem: `gem 'rails', '7.2.2.1'` in `Gemfile`. |
| **Backend** | The server side: rules, calculations, saving data. |

freee-payroll uses **Rails 7.2**. You can see it in two places:

- `Gemfile`: `gem 'rails', '7.2.2.1'`
- `config/application.rb:135`: `config.load_defaults 7.2` (use Rails 7.2's default settings)

---

## 2. MVC: the three roles

Rails organizes code into **MVC**, which stands for **M**odel, **V**iew, **C**ontroller.

| Part | Job | Folder | Everyday analogy (restaurant) |
|---|---|---|---|
| **Model** | Represents data and the rules about it. Usually one model = one database table. | `app/models/` | The kitchen's storeroom + recipes |
| **View** | Produces the output the user sees (HTML) or the data sent back (JSON). | `app/views/` | The plate the food is served on |
| **Controller** | Receives the request, asks models for data, chooses what to send back. | `app/controllers/` | The waiter |

```
Request ──▶ Controller ──asks──▶ Model ──SQL──▶ Database
                │                  ◀──data──
                ▼
              View (HTML or JSON)  ──▶ Response
```

### 🔎 In freee-payroll: MVC is a bit different

In a "classic" Rails app, **views** build full HTML pages. freee-payroll's screens are built with **React** in the browser (`front/`). So:

- Most controllers return **JSON** (the Private API `/api/p/...`), and React displays it.
- The pages themselves use an almost empty HTML view. Look at `app/views/web/app.html.erb:1-6`:

```erb
<%#
  View template for single root
  **DO NOT ADD OTHER ELEMENTS EXCEPT `#app`**
%>

<div id="app"></div>
```

React later fills that `<div id="app">` with the whole screen (Lesson 16 explains how).

And the controller that shows the "holidays" page, `app/controllers/web/holidays_controller.rb:1-8`:

```ruby
# typed: true
class Web::HolidaysController < Web::BaseController
  include ErrorHandleable

  def index
    render template: 'web/app'
  end
end
```

The controller does almost nothing. It just renders the empty `web/app` view. The real screen logic is in the frontend, and the data comes from Private API controllers.

> **Takeaway:** In freee-payroll, **M** and **C** are the heart of the backend. **V** is mostly JSON (or the tiny `web/app` page).

---

## 3. Convention over configuration

| Term | Meaning |
|---|---|
| **Convention** | An agreed-upon rule everybody follows (e.g. "file names use snake_case"). |
| **Configuration** | Writing settings to tell the program something explicitly. |
| **Convention over configuration** | Rails' philosophy: *if you follow its naming rules, you don't have to configure anything.* Rails guesses the connections from the names. |

Example: one concept, many names, all connected automatically:

| Thing | Name | Rule |
|---|---|---|
| Model class | `Comment` | singular, CamelCase |
| Model file | `app/models/comment.rb` | snake_case of the class |
| Database table | `comments` | plural, snake_case |
| Foreign key in other tables | `comment_id` | singular + `_id` |
| Controller class | `Api::Private::CommentsController` | plural + `Controller` |
| Controller file | `app/controllers/api/private/comments_controller.rb` | namespaces become folders |
| Route | `resources :comments` | plural symbol |

Because of these rules, `class Comment < ApplicationRecord` automatically knows its table is `comments`. Nobody wrote `table_name = 'comments'`.

When a name *doesn't* follow the rule, you must configure it. 🔎 `app/models/attendance/check_status.rb:5-8`:

```ruby
  has_many :attendance_check_results,
           class_name: 'Attendance::CheckResult',
           foreign_key: :attendance_check_status_id,
           dependent: :delete_all
```

The association name `attendance_check_results` doesn't match a class called `AttendanceCheckResult`, so `class_name:` and `foreign_key:` say explicitly which class and column to use. Every time you see `class_name:`, `foreign_key:`, or `self.table_name =`, it means "the convention didn't fit, so we configure." (Associations are Lesson 12.)

---

## 4. Autoloading and Zeitwerk: file path ⇄ class name

| Term | Meaning |
|---|---|
| **Constant** | In Ruby, class and module names are constants (they start with a capital letter). |
| **Autoloading** | Rails loads a class's file *automatically* the first time you use the class, so you never write `require 'app/models/comment'`. |
| **Zeitwerk** | The name of Rails' autoloader. It requires that **the file path matches the class name exactly**. |
| **Namespace** | A module that groups classes: `Payroll::Salaries::UnpublishService` lives inside `Payroll::Salaries`. |

The rule: take each folder under an "autoload root" (like `app/models/`, `app/services/`, `app/modules/`), CamelCase it, and join with `::`.

| File | Class/module Zeitwerk expects |
|---|---|
| `app/models/comment.rb` | `Comment` |
| `app/models/attendance/check_status.rb` | `Attendance::CheckStatus` |
| `app/services/payroll/salaries/unpublish_service.rb` | `Payroll::Salaries::UnpublishService` |
| `app/controllers/api/private/employee_display_names_controller.rb` | `Api::Private::EmployeeDisplayNamesController` |
| `app/modules/hr_master/infrastructure/repositories/member_repository.rb` | `HrMaster::Infrastructure::Repositories::MemberRepository` |

Note `app/modules/` is itself a root, so `modules` does **not** appear in the class name.

🔎 `config/application.rb:143`: `config.autoload_lib(ignore: %w[assets])` also makes `lib/` autoloadable. That's why `lib/reachable_checker.rb` → `ReachableChecker` can be used anywhere.

> **Reading tip:** When you see a long name like `::HrMaster::Infrastructure::Repositories::MemberRepository` in code, you can *instantly* find its file: lowercase + snake_case + slashes. The leading `::` just means "start from the top level namespace."

---

## 5. Environments

| Term | Meaning |
|---|---|
| **Environment** | A "mode" the app runs in, with its own settings and database. |
| `development` | On your own computer, while coding. Shows detailed errors. |
| `test` | While running automated tests (RSpec). Uses a separate test database that gets wiped. |
| `staging` | A shared server that copies production, for final checks before release. |
| `production` | The real service customers use. |

🔎 `config/environments/` has exactly these four files: `development.rb`, `test.rb`, `staging.rb`, `production.rb`.

Code often behaves differently per environment. Real example from `lib/reachable_checker.rb` (you'll study it in Lesson 19):

```ruby
raise if Rails.env.development? || Rails.env.test?
```

"If we're in development or test, crash loudly; in production, just report and continue."

Other global settings in `config/application.rb`:

- line 150: `config.time_zone = 'Tokyo'`: all times are handled in Japan time
- line 188: `config.i18n.default_locale = :ja`: messages are Japanese by default

---

## 6. The folders of `app/` in this repo

freee-payroll mixes **two architectures** (from `README.md` and `docs/knowledge/general/architecture/overview-backend.md`):

### 6.1 "Rails-extended" (everything in `app/` except `app/modules/`)

| Folder | Role | Standard Rails? |
|---|---|---|
| `app/controllers/` | HTTP: receive requests, return responses | ✅ standard |
| `app/models/` | ActiveRecord models (DB tables), about 600 files | ✅ standard |
| `app/views/` | Templates (`.erb`, `.jbuilder`) | ✅ standard |
| `app/helpers/` | Helper methods for views | ✅ standard |
| `app/jobs/` | Background jobs (ActiveJob) | ✅ standard |
| `app/mailers/` | Sending emails | ✅ standard |
| `app/services/` | Business logic, "do one task" classes | ➕ freee addition |
| `app/repositories/` | Fetch/save data | ➕ freee addition |
| `app/values/` | Value objects | ➕ freee addition |
| `app/serializers/` | Turn objects into JSON | ➕ (gem) |
| `app/presenters/` | Prepare data for display (e.g. PDFs) | ➕ freee addition |
| `app/decorators/` | Wrap objects to add display methods | ➕ freee addition |
| `app/validators/`, `app/observers/`, `app/resque_jobs/` | Custom validators, observers, legacy jobs | ➕ |

### 6.2 Layered architecture (`app/modules/<business_domain>/`)

```
app/modules/payroll/
├── application/services           # use cases ("update the daily-prorate setting")
├── domain/models                  # business objects, no database knowledge
├── infrastructure/repositories    # reading/writing the DB
├── infrastructure/query_services  # read-only queries
└── infrastructure/factories       # build domain objects from DB records
```

Domains that exist today (`ls app/modules`): admin, ai_shift, application, attendance, common, e_application, egov, elearning, employee_evaluation, employee_master, healthcare, hr_master, hr_portal, nenmatsu_chosei, payroll, period_expansion.

You'll learn these layers deeply in Lessons 17–18. For now: **controllers always stay in `app/controllers/`** (even for `app/modules` code), and **ActiveRecord models always stay in `app/models/`**.

---

## 7. The life of a request (high level)

Let's follow a *real* request: the frontend wants the display name of employee #1.

```
GET /api/p/employee_display_names/1
```

### Step 1: Middleware

| Term | Meaning |
|---|---|
| **Middleware** | Small programs that every request passes through *before* reaching your controller (and every response passes through on the way out). Think of airport security gates. |

In freee-payroll one middleware is **committee**, which checks requests/responses against the API's OpenAPI schema (Lesson 15).

### Step 2: The router

`config/routes.rb` maps URL + method → controller + action. 🔎 `config/routes.rb:236` (inside `namespace :api` and `scope 'p', module: :private`):

```ruby
      resources :employee_display_names, only: [:show]
```

This means: `GET /api/p/employee_display_names/:id` → `Api::Private::EmployeeDisplayNamesController#show`, with `params[:id] = "1"`.

| Term | Meaning |
|---|---|
| **Route** | One rule in the URL map. |
| **Action** | A public method in a controller that handles a route (here, `show`). |
| **params** | A hash-like object holding the URL/query/body values of the request. |

### Step 3: The controller

🔎 `app/controllers/api/private/employee_display_names_controller.rb:1-22`:

```ruby
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

Before `show` runs, the parent classes' **before_action** filters run (login check, permission check...). Then `show`:

1. reads the id from `params` (`show_params[:id].to_i` → `1`)
2. asks a **repository** for the member, limited to the logged-in company (`current_company.id`)
3. **renders JSON**

### Step 4: Repository → Model → Database

The repository (`app/modules/hr_master/infrastructure/repositories/member_repository.rb:44-53`) finds the employee through the **ActiveRecord** model `Employee`, which sends SQL to **MySQL**. It returns a plain domain object `HrMaster::Domain::Models::Member`.

### Step 5: Response

```
200 OK
{ "id": 1, "display_name": "給与太郎" }
```

On the way out, committee checks this JSON matches `employee_display_names_controller.swagger.yaml` (next to the controller file).

### The whole picture

```
Browser
  │  GET /api/p/employee_display_names/1
  ▼
Middleware (committee, auth, ...)
  ▼
Router (config/routes.rb:236)
  ▼
Api::Private::EmployeeDisplayNamesController#show
  │   before_actions (login? permission?)
  ▼
MemberRepository.find_one_by_employee_id(company_id:, employee_id:)
  ▼
Employee (ActiveRecord model) ──SQL──▶ MySQL
  ▼
render json: { id:, display_name: }
  ▼
Browser (React shows the name)
```

> Lessons 14–15 go through each step in detail. Today, just remember the order: **route → controller → (service/repository) → model → DB → JSON**.

---

## 8. The Rails console

| Term | Meaning |
|---|---|
| **Console / REPL** | An interactive prompt where you type Ruby and see the result immediately (like `irb`, but with the whole Rails app loaded). |
| `bin/rails c` | Starts the Rails console. (`bin/rails` is in the repo's `bin/` folder; `c` = `console`.) |

Once your local environment is set up (see the Confluence development guideline linked from `README.md`), you can explore:

```ruby
# (illustrative console session; your data will differ)
company = Company.first
company.employees.count          # how many (not-deleted) employees
company.comments.last            # the newest comment
Comment.column_names             # the columns of the comments table
```

Also useful:

- `bin/rails routes | grep employee_display_names`: lists matching routes
- `bin/rails db:migrate`: applies database changes (Lesson 10)

> ⚠️ Console on your laptop = development database. Never run experiments against staging/production data.

---

## 9. Practice (15 min)

**Practice 1: Zeitwerk.** Write the class name for each file:

1. `app/models/employee_time_clock.rb`
2. `app/controllers/web/holidays_controller.rb`
3. `app/modules/payroll/domain/models/company_daily_prorate_setting.rb`
4. `lib/database_selector.rb`

And the file path for each class:

5. `Api::Private::CommentsController`
6. `Payroll::Infrastructure::Repositories::CompanyDailyProrateSettingRepository`

**Practice 2: find it.** Open `config/routes.rb` and search for `resources :comments`. Which line? Which controller file would handle `GET /api/p/comments`?

**Practice 3: MVC.** For each, write M, V, or C:
a) `app/models/comment.rb` b) `app/views/web/app.html.erb` c) `app/controllers/api/private/comments_controller.rb`

*(Answers: 1 `EmployeeTimeClock`, 2 `Web::HolidaysController`, 3 `Payroll::Domain::Models::CompanyDailyProrateSetting`, 4 `DatabaseSelector`, 5 `app/controllers/api/private/comments_controller.rb`, 6 `app/modules/payroll/infrastructure/repositories/company_daily_prorate_setting_repository.rb`. Practice 2: line 234; `app/controllers/api/private/comments_controller.rb`, action `index`. Practice 3: a) M b) V c) C.)*

---

## 10. Summary

- **Rails** is a backend framework; freee-payroll uses **7.2**.
- **MVC**: Model (data + rules), View (output), Controller (handles request). In freee-payroll the View is mostly **JSON** + a tiny `web/app` page for React.
- **Convention over configuration**: names connect things automatically (`Comment` ⇄ `comments` table ⇄ `comment.rb`).
- **Zeitwerk**: file path must match class name (`app/modules/hr_master/...` → `HrMaster::...`).
- **Environments**: development, test, staging, production.
- **Request lifecycle**: middleware → router → controller (+ before_actions) → service/repository → model → DB → JSON response.
- `bin/rails c` opens the console.

➡️ Now take **Test9.md**.
