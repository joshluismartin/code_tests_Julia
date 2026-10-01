# Lesson 8 — Sorbet: Types for Ruby

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Be able to *read* every Sorbet line in freee-payroll: the `# typed:` comment, `extend T::Sig`, `sig { params(...).returns(...) }`, `T.nilable`, `T::Array[...]`, `T::Struct` with `const`/`prop`, `T::Enum`, `T.let`, `T.must`, `T.type_alias`, and to know what happens when a type is wrong.
**You need:** `~/Josh_Files/freee-payroll/` open in your editor. (Sorbet itself is not installed on your machine outside the repo, so this lesson's practice is *reading and writing on paper*. The answers were checked carefully by reasoning, not by running Sorbet.)

---

## 1. What is a "type"?

| Term | Meaning in plain words | Example |
|---|---|---|
| **Type** | The *kind* of a value: what it is and what you can do with it. | `1` is an `Integer`; `'太郎'` is a `String` |
| **Type error** | Using a value as the wrong kind. | `'abc' + 1` → `TypeError` |
| **Dynamically typed** | Types are only checked *while the program runs*. Plain Ruby is like this. | the error only appears when that line runs |
| **Statically typed** | Types are checked *before running*, by reading the code. | TypeScript (Lesson 24) |
| **Type checker** | A tool that reads code and reports type mistakes without running it. | **Sorbet** for Ruby |
| **Type annotation / signature** | Text you add to code saying what types are expected. | `sig { params(id: Integer).returns(String) }` |
| **nil / nilable** | "may be `nil`". Forgetting that something can be nil is the #1 bug source. | `T.nilable(String)` = String **or** nil |

Plain Ruby:

```ruby
def label(employee_number)
  "No. #{employee_number.upcase}"
end

label('emp001')  # => "No. EMP001"
label(nil)       # 💥 NoMethodError: undefined method 'upcase' for nil — only found when this runs!
```

**Sorbet** lets you write down "`employee_number` is a String" and then **checks it**, both before running and while running.

---

## 2. Two halves of Sorbet

| Part | What it does | When | In freee-payroll |
|---|---|---|---|
| **Static checker** (`srb tc`) | reads all the code and reports type mistakes | on CI, before merge | CircleCI job `ruby_type_check` runs `bundle exec srb typecheck` (`.circleci/config.yml` ~line 498) |
| **Runtime checks** (`sorbet-runtime` gem) | every method with a `sig` checks its arguments and return value **every time it's called** | while the app runs | `gem 'sorbet-runtime'` in `Gemfile` |

### What happens at runtime when a type is wrong? (freee-payroll specific!)

Normally `sorbet-runtime` raises a `TypeError`. freee-payroll changes that in `config/initializers/sorbet.rb:13-19`:

```ruby
  # For example:
  #   HrMaster::Domain::Models::Member.new(employee_id: '1', display_name: '給与太郎')
  config.call_validation_error_handler =
    lambda do |_signature, opts|
      message = "Parameter '#{opts[:name]}': Expected type #{opts[:type]}, got type #{opts[:value].class}"
      ReachableChecker.execute(message: message)
    end
```

It hands the problem to **`ReachableChecker`** (Lesson 7). So:
- in **development / test**, it **raises**, and you see the mistake immediately;
- in **staging / production**, it **reports to Bugsnag and keeps running** (so users aren't broken by a type mistake).

The comment shows the classic mistake: passing `'1'` (a String) where `employee_id` must be an `Integer`.

---

## 3. The `# typed:` sigil (first line of a file)

A **sigil** is a special comment at the top of a file that tells Sorbet **how strictly to check this file**.

| Sigil | Meaning | Count in `app/` (of 5,586 `.rb` files) |
|---|---|---|
| (no sigil) | treated like `false` | ~1,122 |
| `# typed: false` | only very basic checks (syntax, unknown constants) | 115 |
| `# typed: true` | normal type checking of code that has types | **4,161** (most common) |
| `# typed: strict` | every method **must** have a `sig`, and every constant/instance variable must have a known type | 188 (150 of them in `app/modules/`) |

Example: `app/modules/payroll/domain/models/allowance/payroll_setting.rb:1` is `# typed: true`; `app/services/payroll/yearend_adjustments/annual_salary_and_bonus_import/fetch_salary_statement_summary_service.rb:1` is `# typed: strict`.

> 💡 When you *read* code, the sigil tells you how much you can trust the `sig`s. When you *write* new code, follow the level already used in that file, and prefer `true` or `strict` for new files.

---

## 4. Reading a `sig`

To use `sig`, the class must `extend T::Sig` (that's where the `sig` method comes from, Lesson 7). The `sig` goes **directly above** the method it describes.

### 4.1 Anatomy

```ruby
sig { params(company: Company, year: Integer, month: Integer).returns(Result) }
def execute(company:, year:, month:)
```
(`fetch_salary_statement_summary_service.rb:11-12`)

Read it aloud: **"`execute` takes `company` (a `Company`), `year` (an Integer), `month` (an Integer), and returns a `Result`."**

| Piece | Meaning |
|---|---|
| `sig { ... }` | "here is the signature of the next method" |
| `params(name: Type, ...)` | the type of each argument, **by name** (same names as in `def`) |
| `.returns(Type)` | the type of the return value |
| `.void` | "returns nothing useful". Use it instead of `.returns` when the caller shouldn't use the result |
| `sig { returns(Type) }` | a method with **no** arguments: no `params` part |

Long sigs use `do ... end` instead of `{ }` and put one param per line. `app/modules/hr_master/domain/models/member.rb:31-40`:

```ruby
        sig do
          params(
            employee_id: Integer,
            is_deleted: T::Boolean,
            display_name: T.nilable(String),
            employee_number: T.nilable(String),
            user_id: T.nilable(Integer),
            profile_image_url: T.nilable(String),
          ).void
        end
        def initialize(
```
→ "`initialize` takes an Integer `employee_id`, a true/false `is_deleted`, and four values that may be nil; returns nothing."

A getter sig (`member.rb:10-11`):

```ruby
        sig { returns(Integer) }
        attr_reader :employee_id
```

A private-helper sig with `.void` (`fetch_salary_statement_summary_service.rb:35-36`):

```ruby
    sig { params(cps: CompanyPayrollStatement).void }
    def initialize_cps(cps)
```

---

## 5. The type vocabulary

`T` is Sorbet's namespace. Class names (`Integer`, `String`, `Company`, `Date`…) can be used directly as types.

| Type | Means | Example value |
|---|---|---|
| `Integer`, `String`, `Symbol`, `Float`, `Date` | exactly that class (or a subclass) | `1`, `'a'`, `:floor` |
| `Company`, `CompanyPayrollStatement` | an object of that class (models too!) | `Company.find(1)` |
| `T::Boolean` | `true` or `false` (Ruby has no single Boolean class) | `true` |
| `T.nilable(String)` | String **or nil** | `nil`, `'a'` |
| `T::Array[Integer]` | an array whose items are Integers | `[1, 2, 3]` |
| `T::Hash[Symbol, String]` | a hash: Symbol keys, String values | `{ kyokai: 'kyokai' }` |
| `{ id: Integer, status: String }` | a hash with exactly these keys (a "shape") | `{ id: 1, status: 'done' }` |
| `[Member, String]` | a 2-element array: first a Member, then a String (a "tuple") | `[m, 'ok']` |
| `T.any(Integer, String)` | Integer **or** String (a *union*) | `1` or `'1'` |
| `T.untyped` | "anything, don't check". An escape hatch you'll see a lot in older code | anything |
| `T.anything` | also "anything", but safer: you must check before using it | anything |
| `T.proc.params(x: Integer).returns(String)` | a block/lambda type | `->(x) { x.to_s }` |

Real combos:

- `T::Array[T.any(Integer, String)]` (`app/jobs/delete_tasks_job.rb:17`) = an array of items that are each an Integer or a String.
- `T::Hash[Symbol, T.untyped]` (`member.rb:67`) = a hash with symbol keys and values of any type.
- `.returns({ id: Integer, status: String })`. From `app/services/attendance/daily_attendance_tags/imports/enqueue_service.rb:6-8`:

```ruby
      params(company: Company, daily_attendance_tag_import_id: Integer, user_id: Integer).returns(
        { id: Integer, status: String },
      )
```

---

## 6. Inline helpers: `T.let`, `T.must`, `T.type_alias`

### 6.1 `T.let(value, Type)`: declare the type of a variable or constant

Sorbet sometimes can't guess a type (e.g. an empty array `[]` could hold anything). `T.let` says it explicitly **and** checks it at runtime.

```ruby
    messages = T.let([], T::Array[String])
```
(`app/controllers/api/internal/ctax/yearend_adjustment/source_system_controller.rb:344`)

In `# typed: strict` files, constants need `T.let`. `app/models/company_health_insurance_setting_revision.rb:14`:

```ruby
  KumiaiType = T.let({ its: 'its', other: 'other' }, T::Hash[Symbol, String])
```

### 6.2 `T.must(x)`: "I promise this is not nil"

Converts `T.nilable(X)` into `X`. If it **is** nil at runtime, it raises. Real, `fetch_salary_statement_summary_service.rb:22`:

```ruby
          reference_date_name: T.must(cps.company_reference_date_rule).name,
```
`cps.company_reference_date_rule` is typed as maybe-nil. The writer promises it exists here, so `.name` is safe.

> ⚠️ When debugging an error like *"Passed `nil` into T.must"*, the promise was wrong: look at why that value was nil.

### 6.3 `T.type_alias { ... }`: give a long type a short name

`fetch_salary_statement_summary_service.rb:5-6`:

```ruby
  SummaryItem = T.type_alias { { status: String, reference_date_name: String, employees_count: Integer } }
  Result = T.type_alias { T::Array[SummaryItem] }
```
So `.returns(Result)` on line 11 means "returns an array of hashes, each with `status`, `reference_date_name`, `employees_count`".

---

## 7. `T::Struct`: typed data objects

A **`T::Struct`** is a class that just holds typed fields. You write each field once and get `initialize` (with keyword arguments), readers, and type checks for free.

| Keyword | Meaning |
|---|---|
| `const :name, Type` | a **read-only** field (getter only) |
| `prop :name, Type` | a **read-write** field (getter + setter). Rare in freee-payroll: `const` is the norm |
| `default: value` | used when the argument is omitted |
| A `T.nilable(...)` field | may be omitted; it defaults to nil |

Real, `app/modules/application/domain/models/comment.rb:5-17`:

```ruby
      class Comment < T::Struct
        include ActsAsValueObject

        const :id, T.nilable(Integer), default: nil
        const :company_id, Integer
        const :user_id, Integer
        # 画面のURL
        const :url, CommentUrl
        const :page_name, T.nilable(String)
        # コメント本体
        const :comment, T.nilable(String)
        # コメントの作成日時
        const :created_at, ActiveSupport::TimeWithZone
      end
```

What you get, with no hand-written `initialize` (illustrative example):

```ruby
c = Application::Domain::Models::Comment.new(
  company_id: 1, user_id: 2, url: some_comment_url, comment: 'OK', created_at: Time.current,
)
c.company_id   # => 1
c.id           # => nil  (default)
c.company_id = 9   # ❌ NoMethodError: const fields have no setter
Application::Domain::Models::Comment.new(company_id: '1', ...)  # ❌ type error: company_id must be Integer
```

`include ActsAsValueObject` (Lesson 7) adds: validation on creation, `freeze`, value equality.

Another real one, all `T::Boolean`s. `app/modules/payroll/domain/models/allowance/payroll_setting.rb:2-12`:

```ruby
class Payroll::Domain::Models::Allowance::PayrollSetting < T::Struct
  include ActsAsValueObject
  extend T::Sig

  const :is_affect_income_tax, T::Boolean
  const :is_affect_employment_insurance, T::Boolean
  const :is_affect_labor_insurance, T::Boolean
  const :is_affect_fixed_wage, T::Boolean
  # 以下は支給タイミングが賞与の時は false
  const :is_affect_overtime_wage_base, T::Boolean
  const :is_affect_attendance_deduction_base, T::Boolean
```

A mutable `prop` example (rare). `app/modules/elearning/application/services/apply_employee_changes_service.rb:21-28`:

```ruby
        class Group < T::Struct
          prop :id, Integer
          prop :name, String
          prop :code, String
          prop :depth, Integer
          prop :is_disabled, T::Boolean
          prop :parent_group_id, T.nilable(Integer)
        end
```

> Note `app/modules/payroll/domain/models/company_daily_prorate_setting.rb:18-19` uses `const :company_id, T.untyped`: typed, but with "anything allowed". Older/looser code does this; newer code uses real types.

---

## 8. `T::Enum`: a fixed list of allowed values

An **enum** (enumeration) is a type with a fixed set of named values. Real, `app/modules/payroll/domain/models/company_allowance.rb:47-54`:

```ruby
        class Frequency < T::Enum
          enums do
            ThisMonth = new(:this_month)
            Monthly = new(:monthly)
            # 賞与
            Cps = new(:cps)
          end
        end
```

- The values are `Frequency::ThisMonth`, `Frequency::Monthly`, `Frequency::Cps`.
- `Frequency::Monthly.serialize` → `:monthly` (the raw value, e.g. for the DB).
- `Frequency.deserialize(:monthly)` → `Frequency::Monthly`. Used in `app/modules/payroll/infrastructure/factories/company_allowance_factory.rb:83`: `Payroll::Domain::Models::CompanyAllowance::Frequency.deserialize(frequency.to_sym)`.

Why not just use strings? Because `'montly'` (typo) is still a valid String, but `Frequency::Montly` fails immediately.

---

## 9. RBI files, `sorbet/`, and Tapioca (just recognize them)

| Thing | What it is |
|---|---|
| `sorbet/` folder | Sorbet's settings (`sorbet/config`) and type files |
| **RBI file** (`.rbi`) | "Ruby Interface": type info for code that has no sigs (gems, Rails-generated methods) |
| `sorbet/rbi/gems/`, `sorbet/rbi/dsl/` | RBIs generated for gems and for Rails "magic" methods (e.g. `company.employees`) |
| **Tapioca** | the tool that generates those RBIs. Run via `make update-rbi` (`Makefile:175-178`: `bin/tapioca gem` + `bin/tapioca dsl`) |

When `srb tc` complains that a method "does not exist" on a model right after you add an association, the usual fix is regenerating DSL RBIs. You'll rarely edit RBIs by hand.

---

## 10. Practice (15 min), on paper

**Practice 1: read aloud.** Say each in English:
1. `sig { params(employee_ids: T::Array[Integer], company_id: Integer).returns(T::Array[HrMaster::Domain::Models::Member]) }`
2. `sig { params(date: T.nilable(Date)).returns(T::Boolean) }`
3. `sig { void }`

**Practice 2: write the sig.**
1. `def find_one_by_id(company_id:, id:)` returning a `Comment` or nil.
2. `def employee_ids` returning an array of Integers.
3. `def notify!(message:)` returning nothing.

**Practice 3: make a struct.** Write `class Money < T::Struct` with a read-only Integer `amount` and a read-only String `currency` defaulting to `'JPY'`.

**Practice 4: spot the bug.** `sig { params(user_id: Integer).void }` / `def remove(user_id:)` is called as `remove(user_id: params[:user_id])` where `params[:user_id]` is `"12"`. What happens in test? In production?

<details><summary>Practice answers (click)</summary>

1.1 "Takes an array of Integer `employee_ids` and an Integer `company_id`, returns an array of `HrMaster::Domain::Models::Member`." (This is `find_list_by_employee_ids` in `app/modules/hr_master/infrastructure/repositories/member_repository.rb:17-25`.)
1.2 "Takes a `date` that is a Date or nil, returns true/false."
1.3 "Takes no arguments, returns nothing." (Written `sig { void }` for a no-arg method.)

2.1 `sig { params(company_id: Integer, id: Integer).returns(T.nilable(Comment)) }`
2.2 `sig { returns(T::Array[Integer]) }`
2.3 `sig { params(message: String).void }`

3.
```ruby
class Money < T::Struct
  const :amount, Integer
  const :currency, String, default: 'JPY'
end
```

4. `"12"` is a String, not an Integer, so the runtime sig check fails. freee-payroll's `config/initializers/sorbet.rb` sends it to `ReachableChecker`: in **test** it **raises** (the test fails), in **production** it notifies Bugsnag and the method still runs. Fix: `params[:user_id].to_i`.
</details>

---

## 11. Code-reading drill (5 min)

Open `app/services/attendance/daily_attendance_tags/imports/enqueue_service.rb` and, for each of the three `sig`s (lines 5–9, 28–32, 44–46), say the argument types and the return type in English. Then answer: why can line 25 say `T.must(daily_attendance_tag_import.id)`?
*(Because `id` is typed as possibly nil, since a record that isn't saved yet has no id. By this point the record surely exists in the database, so the author asserts it isn't nil.)*

---

## 12. Summary

- A **type** is the kind of a value. **Sorbet** adds types to Ruby: the **static checker** (`srb tc`, CI job `ruby_type_check`) plus **runtime checks** (`sorbet-runtime`).
- In freee-payroll, a runtime type mismatch goes to **ReachableChecker**: it raises in dev/test and is reported (but not fatal) in production.
- `# typed: false/true/strict` = how strict the file is. `true` is most common; `strict` requires sigs everywhere.
- `extend T::Sig` + `sig { params(a: A).returns(R) }` / `.void`, directly above the method.
- Vocabulary: `T::Boolean`, `T.nilable(X)`, `T::Array[X]`, `T::Hash[K, V]`, hash shapes `{ id: Integer }`, `T.any(A, B)`, `T.untyped`.
- `T.let` declares, `T.must` asserts non-nil, `T.type_alias` names a long type.
- `T::Struct` with `const` (read-only, the norm) / `prop` (read-write) / `default:`; `T::Enum` for fixed value lists (`serialize` / `deserialize`).
- RBI files + Tapioca (`make update-rbi`) give types to gems and Rails magic.

➡️ Now take **Test8.md**.
