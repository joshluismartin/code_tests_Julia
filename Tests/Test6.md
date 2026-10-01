# Test 6 — Ruby OOP 1: Classes and Objects

**Rules:** Don't look at Lesson6.md while answering. You may use `irb` only for Part D *after* writing your answer on paper first. Then check `Answers6.md`.
**Total:** 37 points

---

## Part A — Multiple choice (1 point each)

**1.** What is a *class*?
- a) One specific employee record
- b) A blueprint that describes what data objects have and what they can do
- c) A database table
- d) A variable that starts with `@`

**2.** What does `Employee.new('太郎')` do?
- a) Defines a new class
- b) Creates a new object (instance) and calls `initialize('太郎')`
- c) Saves an employee to the database
- d) Calls a class method named `initialize`

**3.** What is `@display_name`?
- a) A local variable
- b) A constant
- c) An instance variable (belongs to one object)
- d) A class method

**4.** What does `attr_reader :name` create?
- a) A method `name` that returns `@name`
- b) A method `name=` that sets `@name`
- c) Both a getter and a setter
- d) A private variable

**5.** Inside `class << self ... end`, every `def` defines:
- a) an instance method
- b) a private method
- c) a class method
- d) a constant

**6.** In freee-payroll, a Service's single public entry point is:
- a) `SomeService.new.run`
- b) `SomeService.execute(...)`
- c) `SomeService.call!`
- d) `SomeService.perform_later`

**7.** Which statement about `private` is true?
- a) Methods above the word `private` become private
- b) Methods below the word `private` can only be called from inside the object/class
- c) `private` deletes the methods
- d) `private` only works in controllers

**8.** In `class Dog < Animal`, `Animal` is the:
- a) subclass
- b) superclass (parent class)
- c) instance
- d) module

**9.** Inside a method, `super` (with no parentheses) calls the parent's method of the same name and passes:
- a) nothing
- b) the same arguments the current method received
- c) only the first argument
- d) `self`

**10.** An instance variable that was never assigned has the value:
- a) `0`
- b) `''`
- c) `nil`
- d) it raises an error

---

## Part B — Fill in the blanks (1 point each blank)

**11.** The special method that runs automatically when you call `.new` is named ______. It is also called the ______.

**12.** To call a setter from inside an object you must write `______.note = 'x'`; otherwise Ruby creates a new ______ variable.

**13.** `attr_accessor :note` creates both a ______ and a ______.

**14.** If you can't find where a controller method is defined, you should walk up the ______ chain (the `<` part) to its parent classes.

---

## Part C — Fill in the missing code (2 points each)

**15.** Complete the class so the last line prints `"EMP001: 太郎"`.

```ruby
class Employee
  ______ :num, :name

  def ______(num, name)
    @num = num
    ______ = name
  end

  def label
    "#{num}: #{name}"
  end
end

puts Employee.new('EMP001', '太郎').label
```

**16.** Make `execute` a class method so `PingService.execute` returns `'pong'`.

```ruby
class PingService
  ______ ______ ______
    def execute
      'pong'
    end
  end
end
```

**17.** Complete `Manager` so `Manager.new('花子', '給与').name` returns `'花子'`.

```ruby
class Staff
  attr_reader :name
  def initialize(name)
    @name = name
  end
end

class Manager ______ Staff
  attr_reader :team
  def initialize(name, team)
    ______(name)
    @team = team
  end
end
```

---

## Part D — Write the code (3 points each)

**18.** Write a class `PaySlip` with:
- `initialize(amount)` storing the amount
- a read-only `amount`
- an instance method `with_tax` returning `amount * 110 / 100`
- a **private** method `currency` returning `'JPY'`, and a public method `label` returning e.g. `"1000 JPY"`.

**19.** Write a service class `CompanyNameService` in freee-payroll style: one public class method `execute(company_id:)` inside `class << self`, which returns the string `"company-#{company_id}"`, plus one **private class method** `prefix` returning `'company'` that `execute` uses.

---

## Part E — Answer in a sentence or two (2 points each)

**20.** What is the difference between an instance method and a class method? Give one example call of each.

**21.** Why does freee-payroll prefer `attr_reader` over `attr_accessor` for things like `Member`?

---

## Part F — Read real freee-payroll code (4 points)

**22.** This is `lib/hr_errors/system_error.rb:28-32`:

```ruby
  class UnauthorizedError < HrErrors::SystemError
    def initialize(message = DEFAULT_MESSAGE, **args)
      super(message, severity: :warning, status: :forbidden, system_message: '現在のプランでは使えない機能を使おうとした', **args)
    end
  end
```

Answer:
- a) What is the parent class of `UnauthorizedError`? (1)
- b) Is `initialize` here overriding something? What? (1)
- c) What does the `super(...)` line do? (1)
- d) What HTTP status will this error always use? (1)
