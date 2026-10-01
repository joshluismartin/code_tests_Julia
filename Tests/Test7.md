# Test 7 — Ruby OOP 2: Modules, Namespaces, Keyword Arguments, Errors, Freeze

**Rules:** Don't look at Lesson7.md while answering. Write answers on paper first; you may check Part D in `irb` afterwards. Then check `Answers7.md`.
**Total:** 38 points

---

## Part A — Multiple choice (1 point each)

**1.** Which is true about a Ruby **module**?
- a) You create objects from it with `.new`
- b) It can be used as a namespace and as a mixin, but cannot be instantiated
- c) It is the same as a database table
- d) It must inherit from `ApplicationRecord`

**2.** What does `::` in `Payroll::Domain::Models` mean?
- a) Comment
- b) "Look inside": the thing on the right is inside the thing on the left
- c) Divide
- d) Call a method on nil safely

**3.** What does a **leading** `::` mean, as in `::HrErrors::UserError`?
- a) The class is private
- b) Look up the name starting from the top level
- c) It's a Sorbet type
- d) It's a typo

**4.** `include SomeModule` makes the module's methods available as:
- a) class methods
- b) instance methods
- c) constants
- d) private constants

**5.** `extend T::Sig` is written so that the class can use which method?
- a) `validates`
- b) `sig`
- c) `freeze`
- d) `execute`

**6.** In an `ActiveSupport::Concern`, code inside `included do ... end` runs:
- a) only in tests
- b) inside the including class at the moment it's included
- c) never
- d) every time a method is called

**7.** `def f(company_id:)` defines `company_id` as:
- a) a positional argument
- b) a required keyword argument
- c) an optional keyword argument
- d) a constant

**8.** `super(message, severity:, original_exception:)` passes `severity:` meaning:
- a) `severity: nil`
- b) `severity: severity` (value taken from the local variable with the same name)
- c) a syntax error
- d) `severity: true`

**9.** Which error class in freee-payroll means "the user did something invalid", with a 4xx status by default?
- a) `HrErrors::SystemError`
- b) `HrErrors::UserError`
- c) `StandardError`
- d) `FrozenError`

**10.** The code in an `ensure` section runs:
- a) only if there was an error
- b) only if there was no error
- c) always, error or not
- d) never in production

**11.** What happens when you call `<<` on a frozen array?
- a) The item is added
- b) `FrozenError` is raised
- c) Returns `nil` silently
- d) The array is copied

**12.** How should freee-payroll code report an unexpected error without crashing?
- a) `Rails.logger.error(e)`
- b) `puts e`
- c) `Omega::Bugcap.notify(e)`
- d) ignore it

---

## Part B — Fill in the blanks (1 point each blank)

**13.** Rails loads classes automatically from their file paths using a tool called ______. So the file `app/modules/hr_master/domain/models/member.rb` must define the constant ______.

**14.** `def f(*args)` collects extra positional arguments into an ______; `def f(**opts)` collects extra keyword arguments into a ______.

**15.** To hide a constant so only its own class can use it, write `______ :MY_CONSTANT`.

**16.** Inside `rescue`, writing `raise` with no arguments ______ the same error.

---

## Part C — Fill in the missing code (2 points each)

**17.** Complete the nested namespace so the full class name is `Payroll::Infrastructure::Repositories::DemoRepository`:

```ruby
______ Payroll
  ______ Infrastructure
    module ______
      class DemoRepository
      end
    end
  end
______
```

**18.** Fill in so that `a` is an instance method and `b` is a class method:

```ruby
module Hello
  def hi = 'hi'
end

class A
  ______ Hello
end

class B
  ______ Hello
end

A.new.hi  # => "hi"
B.hi      # => "hi"
```

**19.** Complete the method so it catches `ActiveRecord::RecordNotFound` and re-raises it as a 404 `UserError`:

```ruby
def find_import(company, id)
  company.daily_attendance_tag_imports.find(id)
______ ActiveRecord::RecordNotFound
  ______ ::HrErrors::UserError.new('存在しません。', ______: :not_found)
end
```

---

## Part D — Write the code (3 points each)

**20.** Write a method `find_one_by_employee_id(company_id:, employee_id:, include_deleted: false)` that returns the string `"company=1 employee=5 deleted=false"` for `find_one_by_employee_id(company_id: 1, employee_id: 5)`. Then write the call using the **keyword shorthand** if local variables `company_id = 1` and `employee_id = 5` already exist.

**21.** Write a custom error class `InvalidPeriodError` (inherits from `StandardError`) and a method `check_month(month)` that raises it with the message `'month must be 1..12'` when month is outside 1..12. Then write a `begin/rescue/ensure` that calls `check_month(13)`, prints the message, and always prints `'checked'`.

---

## Part E — Answer in a sentence or two (2 points each)

**22.** You see `EmployeeMaster::MassUpdates::BulkProcesses::DeleteBulkProcessService` in some code. Explain how you'd find the file.

**23.** Why does freee-payroll prefer keyword arguments (`company_id:`) over positional arguments for services and repositories?

---

## Part F — Read real freee-payroll code (4 points)

**24.** `lib/acts_as_value_object.rb:16-26`:

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

- a) Is `ActsAsValueObject` a class or a module? Can you call `ActsAsValueObject.new`? (1)
- b) When a class does `include ActsAsValueObject`, what happens when you create an instance of it? Describe the three steps of `initialize`. (2)
- c) What does `super` (with no parentheses) pass along here? (1)
