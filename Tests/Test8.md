# Test 8 — Sorbet: Types for Ruby

**Rules:** Don't look at Lesson8.md while answering. Then check `Answers8.md`.
**Total:** 37 points

---

## Part A — Multiple choice (1 point each)

**1.** What is a *type checker*?
- a) A tool that formats code
- b) A tool that reads code and reports values used as the wrong kind, without running it
- c) A database migration
- d) A test framework

**2.** Plain Ruby (without Sorbet) is:
- a) statically typed
- b) dynamically typed: type errors show up only when the line runs
- c) not typed at all; there are no types
- d) typed by TypeScript

**3.** Which sigil makes Sorbet require a `sig` on **every** method in the file?
- a) `# typed: false`
- b) `# typed: true`
- c) `# typed: strict`
- d) `# frozen_string_literal: true`

**4.** Which `# typed:` level is the most common in freee-payroll's `app/`?
- a) `false`
- b) `true`
- c) `strict`
- d) `ignore`

**5.** What must a class do before it can write `sig { ... }`?
- a) `include ActiveModel::Validations`
- b) `extend T::Sig`
- c) `require 'sorbet'`
- d) inherit from `T::Struct`

**6.** `T.nilable(String)` means:
- a) always nil
- b) a String or nil
- c) an empty string
- d) a String that is frozen

**7.** Which type means "true or false"?
- a) `Boolean`
- b) `T::Boolean`
- c) `T.bool`
- d) `TrueClass`

**8.** `.void` in a sig means:
- a) the method raises an error
- b) the method returns nothing useful
- c) the method is private
- d) the method has no arguments

**9.** What does `T.must(x)` do?
- a) Converts x to a String
- b) Asserts x is not nil (raises if it is) and narrows the type from nilable to non-nilable
- c) Makes x required in a form
- d) Freezes x

**10.** In a `T::Struct`, what's the difference between `const` and `prop`?
- a) `const` fields are read-only; `prop` fields also get a setter
- b) `const` is for constants, `prop` is for methods
- c) They are identical
- d) `prop` is required, `const` is optional

**11.** In freee-payroll **production**, if a method is called with an argument of the wrong type (a runtime sig violation), what happens?
- a) The app crashes for everyone
- b) Nothing at all
- c) `ReachableChecker` reports it to Bugsnag and the code continues
- d) The database is rolled back

**12.** What command regenerates the RBI files for gems and Rails DSL methods?
- a) `make generate-root-swagger`
- b) `make update-rbi` (runs Tapioca)
- c) `yarn lint`
- d) `bin/rspec`

---

## Part B — Fill in the blanks (1 point each blank)

**13.** The static check is run on CI by the job named `ruby_type_check` with the command `bundle exec srb ______`.

**14.** `T::Array[Integer]` means an ______ whose items are all ______.

**15.** A `T::______` is a type with a fixed set of named values, and `.deserialize(:monthly)` turns a raw value back into one of them.

**16.** `T.______ { T::Array[SummaryItem] }` gives a long type a short name like `Result`.

---

## Part C — Fill in the missing code (2 points each)

**17.** Complete the sig: takes an Integer `company_id` and an Integer `employee_id`, returns a `HrMaster::Domain::Models::Member`.

```ruby
sig { ______(company_id: ______, employee_id: Integer).______(HrMaster::Domain::Models::Member) }
def find_one_by_employee_id(company_id:, employee_id:)
```

**18.** Complete the struct: read-only Integer `company_id`, read-only String-or-nil `note`, read-only Integer `id` that may be nil and defaults to nil.

```ruby
class Memo < T::Struct
  ______ :company_id, Integer
  const :note, ______(String)
  const :id, T.nilable(Integer), ______: nil
end
```

**19.** Complete so `messages` is declared as an array of Strings:

```ruby
messages = T.______([], T::Array[______])
```

---

## Part D — Write the code (3 points each)

**20.** Write a service in freee-payroll style with Sorbet:
- `# typed: true` sigil
- class `GreetingService` with `class << self` and `extend T::Sig`
- a sig + `execute(name:, times:)` where `name` is a String, `times` is an Integer, returning a String (e.g. `name * times`).

**21.** Write a `T::Enum` called `RoundType` with three values `Floor`, `Round`, `Ceil` whose serialized values are `:floor`, `:round`, `:ceil`. Then write the expression that converts the string `'ceil'` into `RoundType::Ceil`.

---

## Part E — Answer in a sentence or two (2 points each)

**22.** What does Sorbet's *runtime* checking add that the *static* checker (`srb tc`) alone doesn't?

**23.** Why is `T.untyped` considered an "escape hatch", and why should new code avoid it?

---

## Part F — Read real freee-payroll code (4 points)

**24.** `app/services/payroll/yearend_adjustments/annual_salary_and_bonus_import/fetch_salary_statement_summary_service.rb:1-12`:

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
```

- a) How strict is type checking in this file, and what does that require? (1)
- b) In plain English, what does `execute` take and return? Expand `Result` fully. (2)
- c) Give an example of one valid return value. (1)
