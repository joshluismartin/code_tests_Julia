# Lesson 7 — Ruby OOP 2: Modules, Namespaces, Keyword Arguments, Errors, Freeze

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Read long names like `::Payroll::Infrastructure::Repositories::CompanyDailyProrateSettingRepository`, understand `include`/`extend`, read method signatures full of `key:` arguments, and follow `raise` / `rescue` / `ensure` error handling, the way freee-payroll does it.
**You need:** `irb`, and `~/Josh_Files/freee-payroll/` open in your editor.

---

## 1. Modules

A **module** looks like a class, but **you cannot make objects from it** (`MyModule.new` is an error). Modules have two jobs:

1. **Namespace**: a "folder" for names, so two things with the same name don't clash.
2. **Mixin**: a bag of methods you can plug into classes.

| Term | Meaning in plain words |
|---|---|
| **Module** | A container of methods/constants/classes. It can't be instantiated. |
| **Namespace** | A "family name" that groups related classes, e.g. `Payroll::...`, `HrMaster::...`. |
| **Mixin** | A module whose methods get "mixed into" a class with `include` or `extend`. |
| **Constant** | A name starting with a Capital letter. Class and module names are constants too. |
| **Scope resolution operator `::`** | "look inside". `A::B` = "the thing called `B` inside `A`". |

---

## 2. Modules as namespaces

freee-payroll has two classes both named `CompanyDailyProrateSetting`:

- the **ActiveRecord model** `CompanyDailyProrateSetting` (`app/models/company_daily_prorate_setting.rb:2`)
- the **domain model** `Payroll::Domain::Models::CompanyDailyProrateSetting` (`app/modules/payroll/domain/models/company_daily_prorate_setting.rb`)

The namespace (`Payroll::Domain::Models`) keeps them apart. The full name is like a full address: *Japan › Tokyo › Shinagawa › building*.

### 2.1 Two ways to write a namespaced class — you must read both

**Nested style** (open each module one by one). Real file `app/modules/payroll/infrastructure/repositories/company_daily_prorate_setting_repository.rb:1-5`:

```ruby
module Payroll
  module Infrastructure
    module Repositories
      class CompanyDailyProrateSettingRepository
        class << self
```

**Compact style** (one line with `::`). Real file `app/modules/payroll/domain/models/allowance/payroll_setting.rb:2`:

```ruby
class Payroll::Domain::Models::Allowance::PayrollSetting < T::Struct
```

Both define a class whose **full name** is the whole path:
`Payroll::Infrastructure::Repositories::CompanyDailyProrateSettingRepository` and `Payroll::Domain::Models::Allowance::PayrollSetting`.

> **Reading tip:** in nested style, count the `module` lines at the top. Joining their names with `::` gives the full name. The file ends with a matching stack of `end`s. That's why some freee-payroll files end with 5–6 `end` lines in a row.

### 2.2 Using a namespaced name

```ruby
::Payroll::Domain::Models::CompanyDailyProrateSetting.new(company_id: company_id, round_type: 'floor')
```
(from `company_daily_prorate_setting_repository.rb:11`)

### 2.3 The leading `::`

A `::` at the **very start** means "start looking from the **top level**", like `/` at the start of a folder path.

| Written | Ruby looks for |
|---|---|
| `HrErrors::UserError` | first near the current namespace, then outward to the top |
| `::HrErrors::UserError` | only from the top level, never a nearby look-alike |

Why use it? If you're inside `module Payroll` and someone later creates `Payroll::HrErrors`, then plain `HrErrors` would suddenly mean the wrong thing. `::HrErrors` is always the global one. freee-payroll uses the leading `::` a lot, e.g. `raise ::HrErrors::UserError.new(...)` in `app/services/attendance/daily_attendance_tags/imports/enqueue_service.rb:14`. **It doesn't change the meaning when reading — just read `::Foo` as `Foo`.**

### 2.4 File name ↔ constant name (Rails autoloading, "Zeitwerk")

Rails loads classes **automatically** from files based on their path. The tool that does this is called **Zeitwerk**. The rule:

- folders → namespaces, `snake_case` → `CamelCase`
- one file defines one constant with the matching name

| File | Must define |
|---|---|
| `app/models/company_daily_prorate_setting.rb` | `CompanyDailyProrateSetting` |
| `app/modules/payroll/domain/models/allowance/payroll_setting.rb` | `Payroll::Domain::Models::Allowance::PayrollSetting` |
| `app/modules/hr_master/domain/models/member.rb` | `HrMaster::Domain::Models::Member` |
| `lib/hr_errors/user_error.rb` | `HrErrors::UserError` |
| `app/controllers/api/private/comments_controller.rb` | `Api::Private::CommentsController` |

(Folders directly under `app/` like `models/`, `services/`, `modules/`, `controllers/` are **root folders** and are *not* part of the name. `lib/` is autoloaded too: `config/application.rb:143` `config.autoload_lib(ignore: %w[assets])`.)

This is your **#1 navigation skill**: see `EmployeeMaster::MassUpdates::BulkProcesses::DeleteBulkProcessService` in code → open `app/services/employee_master/mass_updates/bulk_processes/delete_bulk_process_service.rb`. (If it isn't under `app/services`, try `app/modules`, `app/models`, `lib/`.) Because of autoloading, **you almost never see `require` in app code**.

---

## 3. Constants

- A **constant** is a name starting with a capital letter. By convention, *value* constants are `ALL_CAPS_WITH_UNDERSCORES`.
- They are meant to never be reassigned (Ruby only warns if you do).

Real examples:

```ruby
DEFAULT_DELETED_AT = Time.utc(1000, 1, 1)                              # config/constants.rb:51
KEYS = %i[jun jul aug sep oct nov dec jan feb mar apr may].freeze      # app/values/resident_tax_monthly_amount.rb:3
DEFAULT_MESSAGE = 'エラーが発生しました。 再読み込みをしても解消しない場合は、サポートセンターまでご連絡ください。'.freeze
                                                                       # lib/hr_errors/base.rb:4
```

- A constant inside a class is read from outside with `::`: `ResidentTaxMonthlyAmount::KEYS`.
- **`private_constant`** hides a constant so only the class itself can use it. Real, `app/models/application_record.rb:68-69`:

```ruby
  IGNORE_OMEGA_LEAK_CHECK_MODELS = [PayrollItem]
  private_constant :IGNORE_OMEGA_LEAK_CHECK_MODELS
```

---

## 4. Modules as mixins: `include` and `extend`

```ruby
module Greetable
  def greet
    "Hello, I am #{name}"
  end
end

class Employee
  include Greetable          # greet becomes an INSTANCE method
  attr_reader :name
  def initialize(name) = @name = name
end

Employee.new('太郎').greet   # => "Hello, I am 太郎"

class Registry
  extend Greetable           # greet becomes a CLASS method
  def self.name = 'Registry'
end

Registry.greet               # => "Hello, I am Registry"
```

| Keyword | Module's methods become… | Memory trick |
|---|---|---|
| `include M` | instance methods (`obj.method`) | **in**clude → **in**stances |
| `extend M` | class methods (`Klass.method`) | **ex**tend → the class it**self** |

### 🔎 In freee-payroll

- `include ActiveModel::Validations` in `Member` (`app/modules/hr_master/domain/models/member.rb:8`) gives each member instance `valid?`, `validate!`, `errors`, and the `validates` rules.
- `include ErrorHandleable` in controllers (e.g. `app/controllers/api/private/employee_display_names_controller.rb:3`) plugs in the standard error handling.
- `extend T::Sig` (seen in almost every typed file) adds the class-level method `sig`, which is why you can write `sig { ... }` above methods (Lesson 8). Inside `class << self` you'll see `extend T::Sig` again, so `sig` also works for the class methods defined there.

### 4.1 Preview: `ActiveSupport::Concern`

Rails adds a nicer way to write mixins called a **concern** (folder `app/models/concerns/`, `app/controllers/concerns/`). A concern is a module with two special blocks:

| Block | Meaning |
|---|---|
| `included do ... end` | code that runs **inside the class** at the moment it's included (e.g. `validates ...`, `scope ...`, callbacks) |
| `class_methods do ... end` | methods that become **class methods** of the including class |
| normal `def`s | become instance methods (just like `include`) |

Real, `app/models/concerns/company_rules/bank_account_validations.rb:1-5`:

```ruby
module CompanyRules::BankAccountValidations
  extend ActiveSupport::Concern

  included do
    validates :company_code, numericality: { only_integer: true }, allow_blank: true, length: { is: 10 }
```

It's used by `app/models/company_bank_account_pay_rule.rb:4`: `include CompanyRules::BankAccountValidations`. So `CompanyBankAccountPayRule` gets those `validates` rules as if they were written inside it.

A `class_methods do` example exists at `app/models/concerns/company_reference_date_rule/reimbursed_expense_cps_finder.rb:55`.
And `Company` gets its release-flag methods like `readonly_obsolete_nencho_enabled?` from `include Company::ReleaseFlags` (`app/models/company.rb:9`).

**Reading tip:** when a model uses a method you can't find in its own file, check its `include` lines at the top. The method is probably in one of those concerns.

---

## 5. Arguments deep dive: positional, keyword, default, `*`, `**`

| Kind | Definition | Call | Notes |
|---|---|---|---|
| **Positional** | `def f(a, b)` | `f(1, 2)` | order matters |
| **Positional with default** | `def f(a, b = 10)` | `f(1)` | `b` becomes 10 |
| **Required keyword** | `def f(company_id:)` | `f(company_id: 1)` | name matters, order doesn't; missing → `ArgumentError` |
| **Keyword with default** | `def f(include_deleted: false)` | `f()` or `f(include_deleted: true)` | optional |
| **Splat `*args`** | `def f(*args)` | `f(1, 2, 3)` | collects extra positional args into an **array** |
| **Double splat `**opts`** | `def f(**opts)` | `f(a: 1, b: 2)` | collects extra keyword args into a **hash** |

```ruby
def find_one(company_id:, employee_id:, include_deleted: false)
  "c=#{company_id} e=#{employee_id} deleted=#{include_deleted}"
end

find_one(employee_id: 5, company_id: 1)             # => "c=1 e=5 deleted=false"
find_one(company_id: 1)                              # ❌ ArgumentError (missing keyword: :employee_id)
find_one(company_id: 1, employee_id: 5, foo: 1)      # ❌ ArgumentError (unknown keyword: :foo)
```

> 🏢 **freee-payroll strongly prefers keyword arguments** for services and repositories. For repositories it's even checked by a test (`spec/repository_class_policies_spec.rb`: "Use keyword arguments for public class methods of a Repository"). Keyword args make calls self-explaining: `find_one_by_employee_id(company_id: 1, employee_id: 5)` can't be confused with `(5, 1)`.

Real signature, `app/modules/hr_master/infrastructure/repositories/member_repository.rb:44`:

```ruby
def find_one_by_employee_id(employee_id:, company_id:, include_deleted: false, check_transaction: true)
```
= two required keywords + two optional ones.

### 5.1 Keyword shorthand `f(x:)` (Ruby 3.1+)

If a local variable has the **same name** as the keyword, you can drop the value:

```ruby
company_id = 1
find_one(company_id:, employee_id: 5)   # same as company_id: company_id
```

Real, `lib/hr_errors/user_error.rb:25`:

```ruby
    super(message, severity:, original_exception:, system_message:, user_message: message.dup)
```
= `super(message, severity: severity, original_exception: original_exception, system_message: system_message, user_message: message.dup)`.

### 5.2 Passing a hash on with `**`

`**args` in a *definition* collects keywords; `**args` in a *call* spreads them back out. See `lib/hr_errors/system_error.rb:29-30`:

```ruby
    def initialize(message = DEFAULT_MESSAGE, **args)
      super(message, severity: :warning, status: :forbidden, system_message: '現在のプランでは使えない機能を使おうとした', **args)
```

---

## 6. Errors (exceptions)

| Term | Meaning |
|---|---|
| **Exception / error** | An object that represents "something went wrong". When it's **raised**, normal execution stops immediately. |
| **raise** | Throw an error. |
| **rescue** | Catch an error and handle it, so the program doesn't crash. |
| **ensure** | Code that runs **no matter what**: success or error. Used for cleanup. |
| **Re-raise** | Inside `rescue`, calling `raise` again with no argument sends the same error onward. |
| **Stack trace / backtrace** | The list of "which method called which" at the moment of the error. Read it top-down to find where it started. |
| **`StandardError`** | The parent class of normal errors. Custom errors usually inherit from it (directly or indirectly). |

### 6.1 Basic shape

```ruby
def divide(a, b)
  raise ArgumentError, 'b must not be 0' if b == 0
  a / b
end

begin
  divide(10, 0)
rescue ArgumentError => e       # catch only this kind; put it in variable e
  puts "Oops: #{e.message}"     # => Oops: b must not be 0
ensure
  puts 'always runs'
end
```

- `rescue => e` with no class catches any `StandardError`.
- Inside a `def`, you can skip `begin`: `rescue`/`ensure` can go directly before the method's `end`.

### 6.2 Custom error classes

```ruby
class PayrollNotFixedError < StandardError; end
raise PayrollNotFixedError, '給与明細が確定していません'
```

freee-payroll's main custom errors live in `lib/hr_errors/`:

| Class | Use for | Defaults (`lib/hr_errors/*.rb`) |
|---|---|---|
| `HrErrors::UserError` | the **user** did something invalid (not found, not allowed, bad input) | `status: :bad_request` (400), `notify_error: false` |
| `HrErrors::SystemError` | a **system** problem (bug, broken data, service down) | `status: :internal_server_error` (500), `notify_error: true` |

Both keep a status so the controller can answer with the right HTTP code (Lesson 15).

### 6.3 🔎 In freee-payroll

**Raise with a status**. `app/services/attendance/daily_attendance_tags/imports/enqueue_service.rb:33-42`:

```ruby
    def daily_attendance_tag_import(company, daily_attendance_tag_import_id, user_id)
      daily_attendance_tag_import = company.daily_attendance_tag_imports.find(daily_attendance_tag_import_id)

      if daily_attendance_tag_import.user_id != user_id
        raise ::HrErrors::UserError.new('他のユーザーがアップロードしたファイルを用いたインポートはできません。', status: :not_found)
      end
      daily_attendance_tag_import
    rescue ActiveRecord::RecordNotFound
      raise ::HrErrors::UserError.new('日次勤怠タグインポート情報が存在しません。', status: :not_found)
    end
```
- `find` raises `ActiveRecord::RecordNotFound` if the id doesn't exist.
- The method-level `rescue` catches it and **translates** it into a friendly `UserError` with status 404. Translating errors like this is a very common pattern.

**Rescue, report, continue**. `app/models/employee.rb:700-706`:

```ruby
    begin
      Omega::GRPC.client(:shift).delete_employee_data(
        ::Proto::Shift::DeleteEmployeeDataRequest.new(employee_id: self.id),
      )
    rescue => e
      Omega::Bugcap.notify(e)
    end
```
If calling the shift microservice fails, don't crash: send the error to the monitoring service (`Omega::Bugcap.notify` → Bugsnag) and carry on. (freee-payroll does **not** use `Rails.logger`, a rule in `app/CLAUDE.md`.)

**ensure for cleanup**. `app/services/attendance/docs/create_attendance_books_service.rb:10-20`:

```ruby
  def execute!
    raise ::HrErrors::UserError.new(system_message: '給与明細が存在しません。') if cps.nil?
    ...
    upload_file!
    update_status!
  ensure
    cleanup_file!
  end
```
The temporary file is deleted even if something above raised.

**raise → rescue → re-raise**. `lib/reachable_checker.rb:12-15`:

```ruby
    def execute(company_id: nil, employee_id: nil, message: nil, custom: {})
      raise UnexpectedReachedError.new(message)
    rescue UnexpectedReachedError => error
      raise if Rails.env.development? || Rails.env.test?
```
It raises its own error on purpose (to capture a backtrace), catches it, then **re-raises** (`raise` alone) in development/test so you notice. In production it notifies Bugsnag instead (line 30). You'll put `ReachableChecker.execute(message: '...')` in branches that "should never happen".

---

## 7. `freeze` and immutability

| Term | Meaning |
|---|---|
| **Mutable** | can be changed after creation |
| **Immutable** | can never change after creation |
| **`freeze`** | makes an object immutable; changing it raises `FrozenError` |
| **`frozen?`** | asks whether an object is frozen |

```ruby
names = %w[太郎 花子].freeze
names << '次郎'        # ❌ FrozenError (can't modify frozen Array)
names.frozen?          # => true
```

Why freeze?
- **Constants**: `KEYS = [...].freeze` makes sure nobody accidentally adds to a shared list.
- **Value objects** (Lesson 17): objects like `Member` call `freeze` at the end of `initialize` (`member.rb:64`), so once built, they're guaranteed not to change.
- `lib/acts_as_value_object.rb:22-26` does it for every value object that includes it:

```ruby
  def initialize(*args)
    super
    validate!
    freeze
  end
```
Read this as: "run the normal constructor (`super`), check the validation rules, then lock the object."

**`# frozen_string_literal: true`** (a *magic comment* at the top of some files) freezes every string literal in that file automatically. Only a minority of freee-payroll files have it (~718 of ~5,600 `.rb` files in `app/`).

⚠️ `freeze` is **shallow**: freezing an array doesn't freeze the strings inside it.

---

## 8. Practice (15 min), in `irb`

**Practice 1: namespace.** Define `module Payroll; module Domain; class Money; ...` with `attr_reader :amount` and `initialize(amount:)`. Create one with `Payroll::Domain::Money.new(amount: 100)`. Then define the *compact* style `class Payroll::Domain::Tax; end` and check `Payroll::Domain::Tax.name`.

**Practice 2: include vs extend.** Make a module `Labelable` with `def label = "[#{self}]"`. `include` it in one class and `extend` it in another. Call it both ways.

**Practice 3: keyword arguments.** Write `def find_list(company_id:, include_deleted: false)` returning a string. Call it with and without `include_deleted`, then without `company_id` and read the error message.

**Practice 4: errors.** Write `def fetch(id)` that raises `KeyError, 'not found'` unless `id == 1`. Call it inside `begin/rescue KeyError => e/ensure` and print `e.message` and `'done'`.

**Practice 5: freeze.** `s = 'abc'.freeze; s << 'd'` and read the error class.

<details><summary>Practice answers (click)</summary>

```ruby
module Payroll
  module Domain
    class Money
      attr_reader :amount
      def initialize(amount:)
        @amount = amount
      end
    end
  end
end
Payroll::Domain::Money.new(amount: 100).amount # => 100
class Payroll::Domain::Tax; end                 # works because Payroll::Domain already exists
Payroll::Domain::Tax.name                       # => "Payroll::Domain::Tax"

module Labelable
  def label = "[#{self}]"
end
class A; include Labelable; end
class B; extend Labelable; end
A.new.label  # => "[#<A:0x...>]"
B.label      # => "[B]"

def find_list(company_id:, include_deleted: false) = "#{company_id}/#{include_deleted}"
find_list(company_id: 1)                        # => "1/false"
find_list(company_id: 1, include_deleted: true) # => "1/true"
find_list                                       # ArgumentError: missing keyword: :company_id

def fetch(id)
  raise KeyError, 'not found' unless id == 1
  'ok'
end
begin
  fetch(2)
rescue KeyError => e
  puts e.message   # not found
ensure
  puts 'done'      # done
end

s = 'abc'.freeze
s << 'd'           # FrozenError: can't modify frozen String: "abc"
```
Note: compact style `class Payroll::Domain::Tax` only works if `Payroll::Domain` is **already defined**. Otherwise Ruby says `uninitialized constant Payroll`. In Rails, autoloading defines the parent namespaces from the folder names, which is why the compact style works in the repo.
</details>

---

## 9. Code-reading drill (5 min)

Open `app/services/attendance/daily_attendance_tags/imports/enqueue_service.rb` and answer:
1. What is the full class name, and which file-path rule tells you where it lives?
2. Which method is the public entry point, and what keyword arguments does it take?
3. List every `raise`. Which error class and status does each use?
4. Which line catches an error, and which error does it catch?

(Answers: 1. `Attendance::DailyAttendanceTags::Imports::EnqueueService`, from `app/services/attendance/daily_attendance_tags/imports/enqueue_service.rb`. 2. `execute(company:, daily_attendance_tag_import_id:, user_id:)`. 3. line 14 `UserError` (default 400), line 37 `UserError` `status: :not_found`, line 41 `UserError` `status: :not_found`. 4. line 40, `ActiveRecord::RecordNotFound`.)

---

## 10. Summary

- **Modules** can't be instantiated. They're used as **namespaces** (`Payroll::Domain::Models::...`) and **mixins**.
- Nested (`module A; module B; class C`) and compact (`class A::B::C`) styles mean the same full name. A leading `::` means "from the top level".
- **Zeitwerk** file rule: `app/modules/payroll/domain/models/allowance/payroll_setting.rb` ⇔ `Payroll::Domain::Models::Allowance::PayrollSetting`. Use it to jump from a name to its file.
- `include` → instance methods; `extend` → class methods. **Concerns** add `included do` and `class_methods do`.
- freee-payroll uses **keyword arguments** everywhere (`company_id:`), including the shorthand `f(x:)`, plus `*args` / `**args`.
- `raise` / `rescue => e` / `ensure` / re-`raise`. Use `HrErrors::UserError` (user's fault, 4xx) and `HrErrors::SystemError` (system's fault, 5xx). Report with `Omega::Bugcap.notify`, never `Rails.logger`.
- `freeze` makes objects immutable; value objects freeze themselves.

➡️ Now take **Test7.md**.
