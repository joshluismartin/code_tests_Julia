# Lesson 6 — Ruby OOP 1: Classes and Objects

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Understand what a *class* and an *object* are, and be able to read any `class ... end` block in freee-payroll: `initialize`, `@variables`, `attr_reader`, `self`, class methods (`def self.x` and `class << self`), `private`, inheritance (`<`) and `super`.
**You need:** A terminal with `irb` (type `irb`, press Enter; type `exit` to quit). Keep `~/Josh_Files/freee-payroll/` open in your editor.

---

## 1. Why classes exist

So far (Lessons 3–5) you stored data in variables, arrays and hashes, and wrote methods with `def`.
Now imagine 300 employees. For each one you need a name, an employee number, and behaviour like "show the display name" or "is this employee deleted?".

You *could* use a hash for each one:

```ruby
employee = { id: 1, name: '給与太郎', deleted: false }
```

…but then every piece of code that uses it must remember the key names, and nothing stops someone from writing `employee[:nmae]` (a typo) and getting `nil`.

**Object-oriented programming (OOP)** solves this by bundling **data** and the **methods that work on that data** into one thing.

| Term | Meaning in plain words | Example |
|---|---|---|
| **Object-oriented programming (OOP)** | A style of programming where you model things as *objects* that hold data and know how to do things. | Ruby is an OOP language. |
| **Class** | A **blueprint**. It describes what data an object has and what it can do. Class names use `CamelCase`. | `class Member ... end` |
| **Object / instance** | One actual thing built from the blueprint. "Instance of `Member`" = one member. | `Member.new(employee_id: 1)` |
| **Instantiate** | To create an object from a class (usually with `.new`). | `Member.new(...)` |
| **Method** | A function that belongs to a class/object. | `member.to_h` |
| **Instance method** | A method you call on an *object*. | `member.to_h` |
| **Class method** | A method you call on the *class itself*. | `Company.find(1)` |
| **Instance variable** | A variable that belongs to one object; starts with `@`. Each object has its own copy. | `@employee_id` |
| **State** | The current values of an object's instance variables. | `@is_deleted = true` |
| **Receiver** | The thing on the left of the dot — the object that "receives" the method call. | in `member.to_h`, `member` is the receiver |

> **Everything in Ruby is an object.** `1`, `'hello'`, `nil`, arrays — all objects. That's why you can write `5.times`, `'abc'.upcase`, `nil.to_s`.
> Try in irb: `1.class` → `Integer`, `'hi'.class` → `String`, `nil.class` → `NilClass`.

---

## 2. Your first class

```ruby
class Employee                      # 1. start a class named Employee
  def initialize(name, num)         # 2. runs automatically when you call Employee.new
    @name = name                    # 3. save the argument into an instance variable
    @num = num
  end

  def label                         # 4. an instance method
    "#{@num}: #{@name}"
  end
end                                 # 5. end of class

taro = Employee.new('給与太郎', 'EMP001')   # create an object (instance)
hanako = Employee.new('労務花子', 'EMP002') # another, separate object

taro.label    # => "EMP001: 給与太郎"
hanako.label  # => "EMP002: 労務花子"
```

Line by line:

1. `class Employee` … `end` defines the blueprint. Class names **must** start with a capital letter.
2. `initialize` is a special method name. You never call it yourself — `Employee.new(...)` creates an empty object and then calls `initialize` with the same arguments. This is called the **constructor**.
3. `@name = name` stores the value *inside this object*. The local variable `name` disappears when `initialize` ends, but `@name` lives as long as the object does.
4. `label` can read `@name` and `@num` because they belong to the same object.

| Term | Meaning |
|---|---|
| **Constructor** | The code that sets up a new object. In Ruby, it's the `initialize` method. |
| **Local variable** | A plain variable like `name`. It only exists inside the method where it was created. |
| **Instance variable** | `@name`. It lives inside the object and every instance method of that object can use it. |

⚠️ An instance variable that was never set is `nil` (no error!). `@nmae` (typo) silently gives `nil`. Be careful when reading code — if something is unexpectedly `nil`, check the spelling of the `@variable`.

---

## 3. Reading data from outside: `attr_reader`, `attr_writer`, `attr_accessor`

Instance variables are **private to the object**. From outside you cannot write `taro.@name`. You need a method that returns it:

```ruby
class Employee
  def initialize(name)
    @name = name
  end

  def name        # a "getter" (reader) method
    @name
  end
end

Employee.new('給与太郎').name  # => "給与太郎"
```

This is so common that Ruby has a shortcut:

| Shortcut | What it creates | Meaning |
|---|---|---|
| `attr_reader :name` | `def name; @name; end` | read only (**getter**) |
| `attr_writer :name` | `def name=(v); @name = v; end` | write only (**setter**) |
| `attr_accessor :name` | both | read and write |

```ruby
class Employee
  attr_reader :name          # outside code can read name
  attr_accessor :note        # outside code can read AND change note

  def initialize(name)
    @name = name
  end
end

e = Employee.new('給与太郎')
e.name          # => "給与太郎"
e.note = '要確認' # calls the setter note=
e.note          # => "要確認"
e.name = 'x'    # => NoMethodError (undefined method 'name=') — no writer!
```

> 💡 freee-payroll prefers **`attr_reader`** (read-only). Objects that can't be changed from outside are safer — you'll see this idea again as **immutability** in Lesson 7.
> The one `attr_accessor` example we'll see: `app/jobs/application_job.rb:4` — `attr_accessor :caller_context # 呼び出し元の文脈`.

---

## 4. `self` — "me, the current object"

Inside a method, **`self`** means "the object this method was called on".

```ruby
class Employee
  attr_reader :name

  def initialize(name)
    @name = name
  end

  def greet
    "Hello, #{self.name}"   # self.name  == name  == @name here
  end

  def same_as?(other)
    self.name == other.name
  end
end
```

- Usually you can leave `self.` off when *reading*: `name` means `self.name`.
- You **must** write `self.` when *calling a setter*, because `note = 'x'` alone would create a new local variable instead:

```ruby
def reset_note
  note = nil        # ❌ creates a local variable called note. Object unchanged!
  self.note = nil   # ✅ calls the setter note=
end
```

You'll see `self.deleted_at = ...` in freee-payroll models for exactly this reason (`config/initializers/logical_delete.rb`: `self.deleted_at = Time.zone.now`).

---

## 5. Class methods: methods on the class itself

Some methods don't belong to one object — they belong to the **class**. The most famous Rails example is `Company.find(1)` ("find the company with id 1"): you don't have a company object yet, so you ask the *class*.

There are **two ways** to write a class method. You must recognise both.

### 5.1 `def self.method_name`

```ruby
class Employee
  def self.default_prefix     # self here = the class Employee
    'EMP'
  end
end

Employee.default_prefix   # => "EMP"   (called on the class)
```

Real example — `app/values/resident_tax_monthly_amount.rb:12-14`:

```ruby
  def self.date_to_key(date)
    date.strftime('%b').downcase.to_sym
  end
```
Called like `ResidentTaxMonthlyAmount.date_to_key(Date.new(2026, 6, 1))` → `:jun`.

### 5.2 `class << self` … `end` (VERY common in freee-payroll)

`class << self` opens a block where **every `def` inside is a class method**. It's handy when a class has *only* class methods — which is exactly what freee-payroll's **Services** and **Repositories** are.

```ruby
class GreetingService
  class << self               # everything until the matching `end` is a class method
    def execute(name:)
      "Hello, #{name}"
    end
  end
end

GreetingService.execute(name: '太郎')   # => "Hello, 太郎"
GreetingService.new.execute             # ❌ NoMethodError: it's not an instance method
```

> 🏢 **freee-payroll rule:** a Service class has exactly **one** public class method, `execute`, and no public instance methods. This is checked automatically by `spec/service_class_policies_spec.rb` (you'll study services in Lesson 17). So when you open a new service, expect `class << self` + `def execute(...)`.

| How it's written | Called as | Seen in |
|---|---|---|
| `def self.find_x` | `Klass.find_x` | older code, value objects |
| `class << self` + `def find_x` | `Klass.find_x` | services, repositories, `lib/` modules |
| plain `def to_h` | `object.to_h` | models, value objects, domain models |

---

## 6. `private` — methods for internal use only

Put the word `private` on its own line. Every method **below it** can only be called from *inside* the object.

```ruby
class Employee
  def initialize(name)
    @name = name
  end

  def label
    "[#{normalized_name}]"      # ✅ calling a private method from inside
  end

  private

  def normalized_name
    @name.strip
  end
end

Employee.new(' 太郎 ').label           # => "[太郎]"
Employee.new(' 太郎 ').normalized_name # ❌ NoMethodError: private method 'normalized_name' called
```

`private` also works inside `class << self` to make **private class methods** — helper steps of `execute`:

Real example — `app/services/payroll/yearend_adjustments/annual_salary_and_bonus_import/fetch_salary_statement_summary_service.rb` (lines 8–12 and 28–32):

```ruby
  class << self
    extend T::Sig

    sig { params(company: Company, year: Integer, month: Integer).returns(Result) }
    def execute(company:, year:, month:)
    ...
    private

    sig { params(company: Company, year: Integer, month: Integer).returns(T::Array[CompanyPayrollStatement]) }
    def fetch_cps_list(company, year, month)
      company.company_reference_date_rules.map { |rule| rule.find_or_create_cps_by_pay_date!(year, month) }
    end
```

- `execute` is public → the only thing other code may call.
- `fetch_cps_list` is private → a helper only `execute` uses.
- Ignore the `sig { ... }` lines for now; they are **Sorbet type signatures** (Lesson 8). Also ignore `extend T::Sig` (Lesson 7/8).

**Reading tip:** when you open a big class, find the word `private`. Everything *above* it is the public "menu" of the class. Everything *below* it is kitchen detail you can skip on first read.

---

## 7. Inheritance: `<` and `super`

**Inheritance** lets one class start as a copy of another class and then add or change things.

| Term | Meaning |
|---|---|
| **Inheritance** | "B is a kind of A". B gets all of A's methods automatically. |
| **Superclass / parent class** | The class being inherited from (A). |
| **Subclass / child class** | The class that inherits (B). |
| **Override** | The subclass defines a method with the *same name*, replacing the parent's version. |
| **`super`** | Inside an overriding method: "now also run the parent's version of this method". |

```ruby
class Animal
  def initialize(name)
    @name = name
  end

  def speak
    '...'
  end

  def intro
    "#{@name} says #{speak}"
  end
end

class Dog < Animal         # Dog inherits from Animal
  def speak                # override
    'Wan!'
  end
end

Dog.new('Pochi').intro     # => "Pochi says Wan!"
Dog.superclass             # => Animal
```

### 7.1 `super` — calling the parent's version

```ruby
class Staff
  attr_reader :name

  def initialize(name)
    @name = name
  end
end

class Manager < Staff
  attr_reader :team

  def initialize(name, team)
    super(name)        # run Staff#initialize(name) → sets @name
    @team = team       # then do the extra Manager-only work
  end
end

m = Manager.new('花子', '給与チーム')
m.name  # => "花子"
m.team  # => "給与チーム"
```

Three forms of `super` you'll meet:

| Form | Passes to the parent |
|---|---|
| `super(a, b)` | exactly `a, b` |
| `super()` | nothing |
| `super` (no parentheses) | **the same arguments this method received** |

### 7.2 🔎 Inheritance in freee-payroll

**Controllers** form a chain. Each one adds behaviour for its level:

```ruby
class ApplicationController < ActionController::Base          # app/controllers/application_controller.rb:1
class Api::BaseController < ApplicationController              # app/controllers/api/base_controller.rb:1
class Api::Private::BaseController < Api::BaseController       # app/controllers/api/private/base_controller.rb:1
class Api::Private::EmployeeDisplayNamesController < Api::Private::BaseController  # app/controllers/api/private/employee_display_names_controller.rb:2
```

So `EmployeeDisplayNamesController` automatically has everything the three parents define (login checks, `current_company`, error handling…). When you can't find where a method comes from, **walk up the `<` chain**.

**Errors** also use inheritance — `lib/hr_errors/system_error.rb:28-32`:

```ruby
  class UnauthorizedError < HrErrors::SystemError
    def initialize(message = DEFAULT_MESSAGE, **args)
      super(message, severity: :warning, status: :forbidden, system_message: '現在のプランでは使えない機能を使おうとした', **args)
    end
  end
```

`UnauthorizedError` is a kind of `SystemError`. Its `initialize` calls `super(...)` to reuse the parent's setup, just forcing `status: :forbidden`. (The `**args` and `key:` things are **keyword arguments** — Lesson 7.)

**ActiveRecord models** all inherit from `ApplicationRecord` (Lesson 11):

```ruby
class CompanyDailyProrateSetting < ApplicationRecord   # app/models/company_daily_prorate_setting.rb:2
```

---

## 8. 🔎 In freee-payroll: reading a whole class

Open **`app/modules/hr_master/domain/models/member.rb`**. It's a real class representing a *member* (a person in the company). Here are lines 6–65, with the Sorbet `sig` lines greyed out in your mind:

```ruby
      class Member
        extend T::Sig
        include ActiveModel::Validations

        sig { returns(Integer) }
        attr_reader :employee_id

        sig { returns(String) }
        attr_reader :employee_number

        sig { returns(String) }
        attr_reader :display_name

        sig { returns(T::Boolean) }
        attr_reader :is_deleted

        sig { returns(T.nilable(Integer)) }
        attr_reader :user_id

        sig { returns(String) }
        attr_reader :profile_image_url

        validates :employee_number, length: { maximum: 255 }
        validates :display_name, presence: true, length: { maximum: 255 }

        sig do
          ...
        end
        def initialize(
          employee_id:,
          is_deleted: false,
          display_name: nil,
          employee_number: nil,
          user_id: nil,
          profile_image_url: nil
        )
          @employee_id = employee_id
          @is_deleted = is_deleted
          @profile_image_url = profile_image_url || ''

          if @is_deleted
            @employee_number = ''
            @display_name = '削除済みの従業員'
            @user_id = nil
          else
            @employee_number = employee_number.to_s
            @display_name = display_name.to_s
            @user_id = user_id
          end

          validate!
          freeze
        end
```

What you can now read:

| Line(s) | Meaning |
|---|---|
| `class Member` | Blueprint for a member. |
| six `attr_reader`s | Outside code can *read* these values; there are **no setters**, so nobody can change them. |
| `def initialize(employee_id:, ...)` | Constructor. `employee_id:` (with a colon) = a **required keyword argument**; `is_deleted: false` = keyword argument with a **default**. (Details in Lesson 7.) |
| `@employee_id = employee_id` | Store the argument inside the object. |
| `profile_image_url \|\| ''` | "Use the given URL, or `''` if it's nil" (Lesson 4). |
| `if @is_deleted ...` | A deleted member always shows the name `削除済みの従業員` ("deleted employee"). |
| `validate!` | Run the `validates` rules; raise an error if invalid (from `ActiveModel::Validations`, Lesson 12). |
| `freeze` | Lock the object so its state can never change (Lesson 7). |

How it's used (real test, `spec/controllers/api/private/employee_display_names_controller_spec.rb:12`):

```ruby
HrMaster::Domain::Models::Member.new(employee_id: 1, display_name: '給与太郎', employee_number: 'EMP001')
```

…and then elsewhere `member.display_name` → `"給与太郎"`.

---

## 9. Practice (15 min) — type these in `irb`

**Practice 1: build a class.** Create `Department` with:
- `initialize(name, code)` storing both
- `attr_reader` for both
- instance method `label` returning `"CODE - NAME"`, e.g. `"D01 - 給与部"`

```ruby
d = Department.new('給与部', 'D01')
d.label  # should be "D01 - 給与部"
```

**Practice 2: class method with `class << self`.** Create `TaxService` with a class method `execute(amount:)` that returns `amount * 10 / 100`. Check `TaxService.execute(amount: 1000)` → `100`.

**Practice 3: private.** Add a private method `prefix` to `Department` that returns `'部門'`, and change `label` to `"#{prefix}: #{@code} - #{@name}"`. Confirm `d.prefix` raises `NoMethodError`.

**Practice 4: inheritance.** Create `SalesDepartment < Department` whose `initialize(name)` calls `super(name, 'S00')`. Check `SalesDepartment.new('営業部').label`.

<details><summary>Practice answers (click)</summary>

```ruby
class Department
  attr_reader :name, :code

  def initialize(name, code)
    @name = name
    @code = code
  end

  def label
    "#{prefix}: #{@code} - #{@name}"   # Practice 1 was "#{@code} - #{@name}"
  end

  private

  def prefix
    '部門'
  end
end

class TaxService
  class << self
    def execute(amount:)
      amount * 10 / 100
    end
  end
end

class SalesDepartment < Department
  def initialize(name)
    super(name, 'S00')
  end
end

SalesDepartment.new('営業部').label  # => "部門: S00 - 営業部"
```
</details>

---

## 10. Code-reading drill (5 min)

Open **`app/modules/payroll/infrastructure/repositories/company_daily_prorate_setting_repository.rb`** and answer for yourself:
1. How many classes are defined? (1: `CompanyDailyProrateSettingRepository`)
2. Are its methods class methods or instance methods? (class methods — inside `class << self`)
3. What are the names of its public methods? (`find_one_by_company_id`, `save`)
4. How would you call the first one? (`Payroll::Infrastructure::Repositories::CompanyDailyProrateSettingRepository.find_one_by_company_id(company_id: 1)`)

(The `module Payroll / module Infrastructure / module Repositories` wrapping is the topic of Lesson 7.)

---

## 11. Summary

- A **class** is a blueprint; an **object/instance** is one thing made with `.new`.
- `initialize` is the **constructor**; `@variables` are **instance variables** holding the object's **state**.
- `attr_reader` = getter, `attr_writer` = setter, `attr_accessor` = both. freee-payroll prefers read-only.
- `self` = the current object (or, in a class method, the class). Use `self.x = ...` to call setters.
- **Class methods**: `def self.x` or inside `class << self`. Services/Repositories in freee-payroll use `class << self` and are called like `SomeService.execute(...)`.
- Methods below `private` are internal helpers.
- `class B < A` = **inheritance**; `super` runs the parent's version of the same method. Walk up the `<` chain to find where a method comes from.

➡️ Now take **Test6.md**.
