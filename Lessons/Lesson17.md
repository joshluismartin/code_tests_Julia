# Lesson 17 — Rails-Extended Layers: Services, Repositories, Values, Presenters

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Know *why* freee-payroll adds extra folders to plain Rails (`services/`, `repositories/`, `values/`, `presenters/`, `serializers/`, `decorators/`), recognize each kind of class on sight, and know the rules CI enforces on them.
**You need:** Lessons 3–16 (Ruby, Sorbet, ActiveRecord, controllers, Private API). Open `~/Josh_Files/freee-payroll/` in your editor.

---

## 1. Why extra layers?

In Lesson 9 you learned plain Rails **MVC**: Model, View, Controller. In a small app that's enough. freee-payroll is **huge** (600+ model files, thousands of endpoints), and payroll rules are complicated. If every rule lived in controllers or models, those files would become thousands of lines long. `app/models/employee.rb` is already ~1,800 lines!

So the team added more **layers**.

| Term | Meaning in plain words |
|---|---|
| **Business logic** | The rules of the business. "Overtime is paid at 1.25×", "you can't delete the last working-hours setting". |
| **Layer** | A group of classes with one kind of job. Code in one layer mostly calls the layer "below" it. |
| **Separation of concerns** | Each piece of code has *one* job. The controller handles HTTP, the service handles the rule, the repository handles data. |
| **Responsibility** | "The job" of a class. A class with too many responsibilities is hard to change safely. |
| **Coupling** | How much one piece of code depends on another. *Low coupling* = you can change one without breaking the other. Good. |
| **Policy spec** | A special RSpec test in freee-payroll that checks the *shape* of your code (method names, visibility), not its behavior. If you break the rule, CI fails. |
| **Cop** | One RuboCop rule. `Hr/UsedRailsLogger` is a freee-payroll custom cop. |
| **Legacy** | Old code written before the current rules. It still works, but new code should not copy its style. |

From `README.md` and `docs/knowledge/general/architecture/overview-backend.md`, the **Rails-extended** architecture (everything outside `app/modules/`) looks like this:

| Folder | Job |
|---|---|
| `app/controllers` | HTTP: read params, call something, render a response |
| `app/models` | ActiveRecord models (tables) |
| `app/services` | **Business logic**, one task per class |
| `app/repositories` | Fetch/save data (hide *how* data is stored) |
| `app/serializers` | Turn an object into JSON for a response |
| `app/presenters` | Prepare data for a view/PDF |
| `app/values` | **Value objects**: small immutable data holders |
| `app/decorators` | Wrap an object to add display-friendly methods |

```
 HTTP request
     │
┌────▼─────────────┐
│  Controller       │  "What was asked? Who is asking?"
└────┬─────────────┘
     │ calls .execute(...)
┌────▼─────────────┐
│  Service          │  "Do the business task" (rules, transactions)
└────┬─────────────┘
     │ find_one_by_... / save
┌────▼─────────────┐
│  Repository       │  "Get/save data" (hides ActiveRecord / gRPC)
└────┬─────────────┘
     │
┌────▼─────────────┐
│ Model (ActiveRecord) → MySQL
└──────────────────┘
 Response: Controller → Serializer / plain Hash → JSON
```

> Not every request uses every layer. A simple "show" endpoint can go Controller → Repository directly (you saw `EmployeeDisplayNamesController` do that in Lesson 15).

---

## 2. Services

### 2.1 What a service is

A **Service** is a class that performs **one business task**. Its name is usually a verb phrase + `Service`: `DeleteBulkProcessService`, `EnqueueService`, `UpdateCompanyAllowanceService`.

### 2.2 The rule: only one public class method, `execute`

`app/README.md:13` says (translated): *"A Service class's public methods are only one class method: `execute`. Checked by `spec/service_class_policies_spec.rb`."*

Here is the actual check, `spec/service_class_policies_spec.rb:404-418`:

```ruby
  it 'publicクラスメソッド execute しか持たず、インスタンスメソッドを持たないこと' do
    service_classes =
      (
        Dir.glob('**/*service.rb', base: "#{Rails.application.root}/app/services") +
          Dir.glob('**/*service.rb', base: "#{Rails.application.root}/lib") +
          Dir.glob('**/*service.rb', base: "#{Rails.application.root}/app/modules")
      ).sort.map { |file| file.delete_suffix('.rb').classify.constantize }
    violated_classes =
      service_classes.select do |klass|
        # NOTE: singleton_method_added は extend T::Sig によって追加されるメソッドであるため除外
        (klass.public_methods - Class.public_methods - [:singleton_method_added]) != [:execute] ||
          (klass.public_instance_methods - Class.public_instance_methods).present?
      end
    expect(violated_classes).to contain_exactly(*ignore_list)
  end
```

Read it line by line:

1. The `it` description means: *"has only the public class method execute and no instance methods."*
2. `Dir.glob('**/*service.rb', ...)` finds **every file ending in `service.rb`** in `app/services`, `lib`, and `app/modules`.
3. `.classify.constantize` turns a file path like `payroll/foo_service` into the class `Payroll::FooService`.
4. `klass.public_methods - Class.public_methods` = "public class methods this class *added*". It must be exactly `[:execute]`.
5. `public_instance_methods ... .present?`: if the class has *any* public instance method, it fails.
6. `contain_exactly(*ignore_list)`: the failing classes must be *exactly* the known legacy ones. **A new class that breaks the rule makes CI fail.**

### 2.3 A clean, modern service

`app/services/employee_master/mass_updates/bulk_processes/delete_bulk_process_service.rb:1-19`:

```ruby
# typed: true

class EmployeeMaster::MassUpdates::BulkProcesses::DeleteBulkProcessService
  class << self
    extend T::Sig

    def execute(company:, bulk_process_id:)
      ActiveRecord::Base.transaction do
        # BulkProcessを取得後、そのBulkProcessがcompleteになってしまうことを避ける
        company
          .bulk_processes
          .where(type: BulkProcess.deletable_history_types_for_employee_bulk_updates)
          .where(status: %i[initialized tmp_uploaded failed undone cancel_requested canceled])
          .find(bulk_process_id)
          .destroy_logically!
      end
    end
  end
end
```

What to notice:

- `class << self ... end` opens the **class's own methods** (Lesson 6). So `def execute` inside it is a **class method**, called as `DeleteBulkProcessService.execute(company: c, bulk_process_id: 5)`. No `.new`.
- **Keyword arguments** (`company:`, `bulk_process_id:`). The caller must write the names, which makes calls readable.
- `ActiveRecord::Base.transaction do ... end` wraps the work in a **transaction** (Lesson 13): either everything succeeds or nothing is saved.
- `destroy_logically!` is a **soft delete** (it sets `deleted_at`; see `config/initializers/logical_delete.rb`).

### 2.4 A strict, typed service with private helpers

`app/services/payroll/yearend_adjustments/annual_salary_and_bonus_import/fetch_salary_statement_summary_service.rb:1-41` (shortened in the middle):

```ruby
# typed: strict
# frozen_string_literal: true

class Payroll::YearendAdjustments::AnnualSalaryAndBonusImport::FetchSalaryStatementSummaryService
  SummaryItem = T.type_alias { { status: String, reference_date_name: String, employees_count: Integer } }
  Result = T.type_alias { T::Array[SummaryItem] }

  class << self
    extend T::Sig

    sig { params(company: Company, year: Integer, month: Integer).returns(Result) }
    def execute(company:, year:, month:)
      cps_list = fetch_cps_list(company, year, month)
      cps_list.each { |cps| initialize_cps(cps) }
      ...
    end

    private

    sig { params(company: Company, year: Integer, month: Integer).returns(T::Array[CompanyPayrollStatement]) }
    def fetch_cps_list(company, year, month)
      company.company_reference_date_rules.map { |rule| rule.find_or_create_cps_by_pay_date!(year, month) }
    end
    ...
  end
end
```

- Helper methods go **after `private` inside `class << self`**. They are private class methods, so the policy spec doesn't count them.
- `T.type_alias` gives a long type a short name (`Result`).

### 2.5 You will also see legacy services (don't copy them)

`app/services/payroll/salaries/unpublish_service.rb:1-19`:

```ruby
# typed: true
class Payroll::Salaries::UnpublishService < ::Payroll::Salaries::BaseService
  def initialize(company, auth_company, user, options = {})
    super(
      ...
    )
  end

  def execute
    # 削除する
    @current_cps.company_payroll_statement_scheduled_publish_settings.destroy_all_logically!

    @current_cps.unpublish_new!
  end
end
```

It's called as `UnpublishService.new(company, auth_company, user, year: 2021).execute`: an **instance** method. That breaks the rule, so the class is listed in the spec's `ignore_list` (`spec/service_class_policies_spec.rb:335`). Line 7 of the spec says (translated): *"TODO: remove these in future. Temporary so existing classes don't fail. **Do not add new ones.**"*

A subtler one: `app/services/attendance/daily_attendance_tags/imports/enqueue_service.rb` defines helpers `daily_attendance_tag_import` and `is_importing?` inside `class << self` but **forgot `private`**, so they're public. It's also on the ignore list (line 87).

| You see... | It is... | New code should... |
|---|---|---|
| `class << self` + `def execute(keyword:)` | Modern style ✅ | Look like this |
| `def initialize(...)` + `def execute` (instance) | Legacy | Not copy it |

---

## 3. Repositories

### 3.1 What a repository is

A **Repository** is a class whose job is **getting and saving data**. The caller asks in business words ("find the member for employee 5"). The repository hides *how*: ActiveRecord, a gRPC call to another service, or S3.

Why? If the data moves (for example from our MySQL to the nest-auth microservice), you only change the repository and the callers don't notice. That's **low coupling**.

### 3.2 Naming rules (enforced!)

From `docs/knowledge/general/architecture/repository-naming.md` and `app/README.md:14`:

| Prefix | Meaning | Examples |
|---|---|---|
| `find_one_by_<thing>` | Exactly **one** record, filtered by a property | `find_one_by_id`, `find_one_by_employee_id` |
| `find_one_at_<time>` | One record as of a **point in time** | `find_one_at_pay_year_month` |
| `find_list_by_<thing>` | **Many** records, filtered | `find_list_by_employee_ids`, `find_list_by_url` |
| `find_list_at_<time>` | Many records, as of a time | `find_list_at_entry_date` |
| `find_all` | Many, **no filter** except company | `find_all` |
| `find_all_at_<time>` | All, as of a time | `find_all_at_...` |

The words: **by** = "filtered by a property", **at** = "at a point in time".

The check lives in `spec/repository_class_policies_spec.rb:29-66`. Key parts:

```ruby
  def verify(klass)
    if (klass.public_instance_methods - Class.public_instance_methods).present?
      raise "#{klass}: Use class methods for public methods of a Repository."
    end
    ...
        test_target_methods.include?(el.method_name) && !el.arguments.all? { |a| a.kwarg_type? || a.kwoptarg_type? }
    ...
          raise "Use keyword arguments for public class methods of a Repository. class: #{klass}, methods: ..."
    ...
        case method.to_s
        when /^find_(one|list).*$/
          unless method.to_s.match?(/^find_(one|list)(_by_|_at).*$/)
            raise "...When filtering by property (_by) or time (_at), add it to the method's name..."
          end
        when /^find_all.*$/
          unless method.to_s.match?(/^find_all(_at.*)?$/)
            raise "...Do not add filters to find_all. When filtering by property, create a method named find_list_by_* instead."
          end
        else
          raise "...Use find_one_*, find_list_*, or find_all* instead."
        end
```

So a repository must:

1. Use **class methods** only (no public instance methods).
2. Use **keyword arguments** for every public method (`kwarg` = required keyword, `kwoptarg` = keyword with default).
3. Name every `find...` method as `find_one_by_* / find_one_at* / find_list_by_* / find_list_at* / find_all / find_all_at*`.

Methods that don't start with `find` (`save`, `update`, `create`, `delete`) are allowed; they just need keyword args.

> ⚠️ **Watch out:** the naming doc shows `find_all_employees` as an example, but the regex `^find_all(_at.*)?$` would **reject** that name. When a doc and a policy spec disagree, **the spec is what CI runs**, so trust the spec. (Real code uses plain `find_all`.)

### 3.3 A real, compliant repository

`app/modules/payroll/infrastructure/repositories/company_allowance_repository.rb` (手当マスタ = allowance master). Line 9, then line 45, then line 113:

```ruby
    def find_one_by_id(company_id:, company_allowance_id:)
      company = Company.find(company_id)
      ...
    def find_all(company_id:, frequencies: nil, is_active: nil)
      ...
    def find_list_by_company_id_for_display_order_update_candidate_ids(company_id:)
```

Interesting: `find_all` has optional keyword args (`frequencies: nil`). The spec checks the **name**, not the arguments, so this passes. The *spirit* of the rule is: if filtering is the main point, name it `find_list_by_*`.

Also at line 138, above `update`:

```ruby
    # NOTE: 複数テーブルへの書き込みを行うため、呼び出し元のサービスでトランザクションを張ること
```

"This writes to multiple tables, so **the calling service must open the transaction**." Repositories do the data work; **services own transactions**.

### 3.4 Legacy repositories

`app/repositories/` has only a few files (`application/`, `employee_master/`, `payroll/`). Several are on the repository spec's `ignore_list` (`spec/repository_class_policies_spec.rb:6-27`), for example `Application::CompanyOnboardingTaskRepository`. Most modern repositories live in `app/modules/*/infrastructure/repositories/` (Lesson 18).

---

## 4. Values (value objects)

### 4.1 What a value object is

| Term | Meaning |
|---|---|
| **Value object** | A small object defined *only by its values*. Two value objects with the same values are **equal**. Examples: a date range, an amount of money, a birthdate. |
| **Immutable** | Can't be changed after it's created. To "change" it, you make a new one. |
| **Equality by value** | `YearMonth(2026, 9) == YearMonth(2026, 9)` is `true` even though they're two different objects in memory. |

### 4.2 How freee-payroll writes them: `T::Struct` + `ActsAsValueObject`

`lib/acts_as_value_object.rb:16-26`:

```ruby
module ActsAsValueObject
  extend ActiveSupport::Concern
  extend T::Sig
  include ActiveModel::Validations
  include Kernel

  def initialize(*args)
    super
    validate!
    freeze
  end
```

When you include it in a `T::Struct`:

- `include ActiveModel::Validations` lets you write `validates ...` like in a model (Lesson 12).
- `initialize` calls `super` (T::Struct sets the fields), then `validate!` (**raises** `ActiveModel::ValidationError` if invalid), then `freeze` (makes it **immutable**).
- **Equality by value** comes from `config/initializers/monkey_patch_t_struct.rb:4` (`T::Struct.include(::T::Struct::ActsAsComparable)`), which makes `==` compare field values for *every* `T::Struct` in the app.

Real example: `app/values/birthdate.rb:6-9`

```ruby
class Birthdate < T::Struct
  include ActsAsValueObject

  const :birthdate, T.untyped
```

Usage (from the comment at `app/values/birthdate.rb:21`):

```ruby
Birthdate.new(birthdate: Date.new(2000, 1, 1)).age_at(date: Date.new(2019, 12, 31), based_on_nenrei_keisan_ni_kansuru_horitsu: true)
#   => 20
```

(年齢計算ニ関スル法律 = the Japanese law on age calculation: you legally turn one year older the day *before* your birthday. That's why the flag adds `+ 1.day`.)

A value object with validation (illustrative example, same pattern as the repo's `Payroll::Domain::Models::CompanyDailyProrateSetting`):

```ruby
class RoundType < T::Struct
  include ActsAsValueObject

  const :value, String

  validates :value, inclusion: %w[floor round ceil]
end

RoundType.new(value: 'floor')   # ok, frozen
RoundType.new(value: 'banana')  # raises ActiveModel::ValidationError
RoundType.new(value: 'floor') == RoundType.new(value: 'floor') # => true
```

> Older value objects exist too, e.g. `app/values/resident_tax_monthly_amount.rb`: a plain class with `attr_reader` and a hand-written `==`. Recognize it; new code uses `T::Struct` + `ActsAsValueObject` (rule in `app/modules/README.md`).

---

## 5. Serializers, Presenters, Decorators

These three all **prepare data for output**. Beginners mix them up, so here they are side by side:

| Kind | Input | Output | Folder | Count |
|---|---|---|---|---|
| **Serializer** | An object (model or domain object) | A Hash that becomes **JSON** | `app/serializers` | ~195 |
| **Presenter** | IDs / company | Data arranged for a **view or PDF** | `app/presenters` | ~79 |
| **Decorator** | One object | The same object plus **display-friendly methods** | `app/decorators` (only `wolf/`) | 5 |

### 5.1 Serializer (`ActiveModel::Serializer`, old 0.8 version)

`app/serializers/comment_serializer.rb:3-24`:

```ruby
class CommentSerializer < ActiveModel::Serializer
  attributes :created_at,
             :employee_profile_image_url,
             :user_name,
             :id,
             ...

  def user_name
    options[:authors_by_user_id].dig(object.user_id, :name)
  end

  def created_at
    object.created_at.strftime('%Y-%m-%d %H:%M')
  end
```

- `attributes` lists the JSON keys.
- If a method with the same name exists, the serializer calls it (e.g. `created_at` formats the time).
- `object` = the thing being serialized; `options` = extra data passed in by the controller.

Many modern controllers skip serializers and write `render json: { round_type: setting.round_type }` directly (Lesson 15).

⚠️ `app/README.md:16`: **`EmployeeSerializer` must not be newly referenced** (cop `Hr/UsedEmployeeSerializer`).

### 5.2 Presenter

`app/presenters/employee_list_presenter.rb:2-11` builds data for the **労働者名簿** (legally required employee register) PDF:

```ruby
class EmployeeListPresenter < BasePresenter
  attr_reader :employees

  OPEN_ENDED_VALID_TO = Date.new(9999, 12, 31)

  def initialize(company_id)
    @company_id = company_id
  end

  def build
    @employees = []
```

Used in `app/controllers/pdfs_controller.rb:59-62`:

```ruby
    @presenter = EmployeeListPresenter.new(current_company.id)
    @presenter.build

    respond_to { |format| format.pdf { render pdf_render_options.merge(pdf: '労働者名簿', orientation: 'Portrait') } }
```

The controller creates the presenter, calls `build`, and the PDF template reads `@presenter.employees`.

### 5.3 Decorator

`app/decorators/wolf/flow_log_decorator.rb:3-19` wraps a log record from **Wolf** (the approval-flow microservice):

```ruby
class Wolf::FlowLogDecorator
  include ActiveModel::Model
  include ActiveModel::Serialization

  def initialize(company, log, employee_index:)
    @company = company
    @log = log
    @employee_index = employee_index
  end

  def employee
    @employee_index[@log.user&.id]
  end
```

It "decorates" a raw gRPC object with helpful methods like `employee`, so a serializer can use them.

---

## 6. Rules CI enforces (memorize this table)

From `app/README.md:11-19`:

| Rule | Checked by | What happens if you break it |
|---|---|---|
| Service: only public class method `execute` | `spec/service_class_policies_spec.rb` | RSpec fails in CI |
| Repository: class methods + keyword args + `find_one_by/at`, `find_list_by/at`, `find_all(_at)` naming | `spec/repository_class_policies_spec.rb` | RSpec fails |
| Don't use `Rails.logger` (production logger is disabled) → use `Omega::Bugcap.notify` | Cop `Hr/UsedRailsLogger` | RuboCop fails |
| Don't newly reference `EmployeeSerializer` | Cop `Hr/UsedEmployeeSerializer` | RuboCop fails |
| No direct query chains on AR classes like `Employee.where(...)` (except allow-listed places) → use a model **scope** or a **Repository** | Cop `Freee/QueryFromActiveRecordClass` | RuboCop fails |
| New async jobs go in `app/jobs/` (ActiveJob), not `app/resque_jobs/` | `spec/job_class_policies_spec.rb` | RSpec fails (Lesson 19) |
| Controllers include `ErrorHandleable` (or `Omega::Controller::ErrorHandler`) | `spec/controller_class_policies_spec.rb` | RSpec fails |

> Like services, these specs have `ignore_list`s of legacy offenders. For example, `Api::Private::CompanyAllowancesController` doesn't include `ErrorHandleable` and is ignore-listed at `spec/controller_class_policies_spec.rb:37`. **"It exists in the repo" does not mean "it's allowed for new code."** Check the ignore lists when in doubt.

---

## 7. Putting it together: a full Rails-extended-style request

Endpoint `DELETE /api/p/comments/:id` in `app/controllers/api/private/comments_controller.rb:45-59`:

```ruby
  def destroy
    comment =
      Application::Infrastructure::Repositories::CommentRepository.find_one_by_id(
        company_id: current_company.id,
        id: params[:id].to_i,
      )

    raise HrErrors::UserError.new('コメントの削除権限がありません。', status: :forbidden) if comment.user_id != current_auth_user.id

    ActiveRecord::Base.transaction do
      Application::Infrastructure::Repositories::CommentRepository.delete(comment: comment)
    end

    head :no_content
  end
```

- Repository `find_one_by_id(company_id:, id:)` ✔ naming, ✔ keyword args
- A business check (only the author can delete) raises `HrErrors::UserError` → `ErrorHandleable` turns it into a JSON 403
- Transaction around the write
- `head :no_content` = status 204 with no body

---

## 8. Practice (15 min)

**Practice 1: classify.** Open each file and say whether it's a Service, Repository, Value object, Serializer, Presenter, or Decorator. Then say modern or legacy.
1. `app/services/payroll/salaries/unpublish_service.rb`
2. `app/values/birthdate.rb`
3. `app/decorators/wolf/flow_decorator.rb`
4. `app/modules/payroll/infrastructure/repositories/company_daily_prorate_setting_repository.rb`

**Practice 2: policy judge.** Would these repository class-method names pass `verify`?
`find_one_by_employee_id`, `find_employees`, `find_all_active`, `find_list_at_pay_year_month`, `find_one`, `save`

**Practice 3: find ignore lists.** Open `spec/service_class_policies_spec.rb` and count roughly how many legacy services are ignore-listed (look at where `ignore_list = [` starts and ends).

**Practice 4 (irb).** Run `irb` and type this plain-Ruby imitation of a frozen value object:
```ruby
Point = Struct.new(:x, :y)
a = Point.new(1, 2).freeze
b = Point.new(1, 2)
a == b      # ?
a.x = 5     # ?
```

*(Answers: 1 = Service, legacy (instance `execute`, ignore-listed); 2 = Value object, modern (`T::Struct` + `ActsAsValueObject`); 3 = Decorator; 4 = Repository (modern, in `app/modules`). 2 = pass / fail (doesn't start with find_one/find_list/find_all) / fail (find_all can't have filters in name) / pass / fail (find_one needs `_by_` or `_at`) / pass (not a find method; still needs keyword args). 3 = the list runs from line 8 to line ~400, several hundred classes. 4 = `true` (equal by value); `FrozenError` (can't modify frozen).)*

---

## 9. Summary

- freee-payroll adds **layers** to Rails so each class has **one responsibility**.
- **Service** = one business task. Modern shape: `class << self` → `def execute(keyword:)`; helpers under `private`. Services own **transactions**.
- **Repository** = get/save data. Class methods, keyword args, `find_one_by_* / find_list_by_* / *_at* / find_all`.
- **Value object** = `T::Struct` + `include ActsAsValueObject` → validated on create, frozen, equal by value.
- **Serializer** → JSON; **Presenter** → view/PDF data; **Decorator** → wraps an object with display helpers.
- Rules are enforced by **policy specs** and **cops**; legacy offenders live in **ignore lists**. Don't copy them.

➡️ Now take **Test17.md**.
