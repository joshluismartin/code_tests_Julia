# Lesson 5: Ruby Basics 3 (Blocks and Iteration)

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Read and write *blocks* (`{ |x| ... }` / `do |x| ... end`) and the iteration methods (`each`, `map`, `select`, `find`, `group_by`, `index_by`, `each_slice`, and more) that fill freee-payroll's code. Understand `&:name`, lambdas `->(x) { }`, and `yield`.
**You need:** `irb`.

---

## 1. What is a block?

| Term | Meaning in plain words |
|---|---|
| **Block** | A small chunk of code you hand to a method, so the method can run it (often once per item). |
| **Block parameter** | The name(s) between pipes `\|x\|`: the value the method gives your block each time. |
| **Iterate / iteration** | Going through items one by one. |
| **Iterator** | A method that iterates and runs your block for each item (`each`, `map`, ...). |
| **Collection** | Anything with many items: Array, Hash, Range, database query result. |

Two ways to write a block. They mean the same thing:

```ruby
[1, 2, 3].each { |n| puts n * 10 }     # braces: for one-liners

[1, 2, 3].each do |n|                  # do...end: for multi-line blocks
  doubled = n * 2
  puts doubled * 5
end
```

Read `[1, 2, 3].each { |n| puts n * 10 }` as: "for **each** item, call it `n`, and run `puts n * 10`." It prints `10`, `20`, `30`.

> ⚠️ Small trap: `p [1, 2, 3].map do |n| n * 2 end` prints `#<Enumerator...>` because `do...end` attaches to `p`, not to `map`. With `{ }` it works. That's one reason one-liners use braces.

---

## 2. The core iterators

These live on Arrays, Ranges, Hashes and (in Rails) query results. **Learn these by heart**; they're in almost every file.

| Method | Returns | Example → result |
|---|---|---|
| `each` | the original collection (use it for *side effects*: printing, saving) | `[1, 2, 3].each { \|n\| n * 10 }` → `[1, 2, 3]` |
| `map` | **a new array** of the block's results ("transform each item") | `[1, 2, 3].map { \|n\| n * 10 }` → `[10, 20, 30]` |
| `select` (a.k.a. `filter`) | items where the block is truthy ("keep") | `[1, 2, 3, 4].select { \|n\| n.even? }` → `[2, 4]` |
| `reject` | items where the block is falsy ("drop") | `[1, 2, 3, 4].reject { \|n\| n.even? }` → `[1, 3]` |
| `find` (a.k.a. `detect`) | the **first** matching item, or `nil` | `[1, 2, 3, 4].find { \|n\| n > 2 }` → `3` |
| `any?` | true if **at least one** matches | `[1, 2, 3].any? { \|n\| n > 2 }` → `true` |
| `all?` | true if **every** item matches | `[1, 2, 3].all? { \|n\| n > 0 }` → `true` |
| `none?` | true if **no** item matches | `[1, 2, 3].none? { \|n\| n > 5 }` → `true` |
| `count` | how many match | `[1, 2, 3, 4].count { \|n\| n.even? }` → `2` |
| `sum` | total (optionally of the block's results) | `[1, 2, 3].sum { \|n\| n * 2 }` → `12` |
| `sort_by` | sorted by the block's value | `[3, 1, 2].sort_by { \|n\| -n }` → `[3, 2, 1]` |
| `max_by` / `min_by` | item with the biggest / smallest value | `employees.max_by { \|e\| e.id }` |
| `each_with_index` | gives item **and** position | `%w[x y].each_with_index { \|v, i\| ... }` |
| `times` | repeat N times | `3.times { \|i\| print i }` → prints `012` |

**`each` vs `map`: the #1 beginner confusion.**
- Use `map` when you want a **new list** built from the old one.
- Use `each` when you just want to **do something** with each item; its return value is ignored.

### 2.1 Chaining

Because each method returns a collection, you can chain them left to right:

```ruby
[1, 2, 3, 4].map { |n| n * 2 }.select { |n| n > 4 }.sum
# [2, 4, 6, 8] → [6, 8] → 14
```

### 2.2 Blocks over a Hash get two parameters

```ruby
{ a: 1, b: 2 }.each { |key, value| puts "#{key}=#{value}" }
{ a: 1, b: 2 }.map { |k, v| "#{k}=#{v}" }   # => ["a=1", "b=2"]
```

---

## 3. The `&:method_name` shorthand

When a block only calls **one method on each item**, you can write `&:method_name` instead:

```ruby
employees.map { |e| e.id }    # long form
employees.map(&:id)           # short form: same result
%w[a b].map(&:upcase)         # => ["A", "B"]
[1, 2, 3, 4].reject(&:even?)  # => [1, 3]
```

Read `map(&:id)` as "**map to their ids**." In freee-payroll it's everywhere: `map(&:id)`, `map(&:employee_id)`, `sum(&:amount)`, `index_by(&:id)`, `group_by(&:employee_id)`.

---

## 4. Grouping and reshaping

For these examples, imagine employees with `id`, `name`, `dept`:
`A(1, sales)`, `B(2, dev)`, `C(3, dev)`.

| Method | What it builds | Example → result |
|---|---|---|
| `group_by` | Hash: **key → array of items** with that key | `employees.group_by(&:dept)` → `{ 'sales' => [A], 'dev' => [B, C] }` |
| `index_by` (**Rails only**) | Hash: **key → one item** (for fast lookup by id) | `employees.index_by(&:id)` → `{ 1 => A, 2 => B, 3 => C }` |
| `to_h { }` | Hash from `[key, value]` pairs | `employees.to_h { \|e\| [e.id, e.name] }` → `{ 1 => 'A', 2 => 'B', 3 => 'C' }` |
| `each_slice(n)` | chunks of n items | `(1..7).each_slice(3).to_a` → `[[1, 2, 3], [4, 5, 6], [7]]` |
| `flat_map` | map, then flatten one level | `[[1, 2], [3]].flat_map { \|a\| a.map { \|x\| x * 10 } }` → `[10, 20, 30]` |
| `each_with_object(obj)` | loop while filling an object you pass in | `%i[a b].each_with_object({}) { \|k, h\| h[k] = k.to_s }` → `{ a: 'a', b: 'b' }` |
| `partition` | two arrays: [matching, not matching] | `employees.partition { \|e\| e.dept == 'dev' }` → `[[B, C], [A]]` |
| `tally` | count each value | `%w[a b a].tally` → `{ 'a' => 2, 'b' => 1 }` |
| `reduce` (a.k.a. `inject`) | combine everything into one value | `[1, 2, 3].reduce(0) { \|acc, n\| acc + n }` → `6` |

**Why `index_by` matters:** instead of searching a list over and over (`employees.find { |e| e.id == 7 }`, which is slow if repeated), build a lookup Hash once and then do `by_id[7]`, which is instant. `index_by` comes from Rails (ActiveSupport), so in plain irb use `to_h { |e| [e.id, e] }`.

**Why `each_slice` matters:** databases and other services limit how many ids you can send at once. `each_slice(1000)` processes 1000 at a time, a pattern required by the team's memory rules for big data (`docs/knowledge/general/coding-standards/memory-optimization.md`).

---

## 5. Lambdas: blocks you can store in a variable

A **lambda** is a block saved as a value, so you can pass it around and call it later.

```ruby
double = ->(x) { x * 2 }   # "stabby lambda" syntax
double.call(4)   # => 8
double.(4)       # => 8     (short form)
double[4]        # => 8     (another short form)
```

You'll see lambdas mostly in Rails **scopes** (Lesson 12): `scope :latest, -> { order(date: :desc) }`, where `->` with no parameters is a lambda that takes nothing.

(`lambda { |x| ... }` is the older spelling of the same thing.)

---

## 6. `yield`: how a method runs *your* block

Inside a method, `yield` means "**run the block the caller gave me**" (and pass it values).

```ruby
def twice
  yield 1
  yield 2
end

twice { |n| puts "got #{n}" }
# got 1
# got 2
```

A very common pattern is "**wrap** the block with setup/cleanup":

```ruby
def with_log
  puts 'start'
  result = yield       # run the caller's block, keep its value
  puts 'end'
  result
end

with_log { 42 }   # prints start, end → returns 42
```

freee-payroll uses this style for things like "run this block on the read-replica database" or "run this block without the leak check": `DatabaseSelector.connected_to_reader { ... }`, `Authz.without_employee_leakcheck do ... end`, `ActiveRecord::Base.transaction do ... end`. When you see `SomeThing.do_x { ... }` or `do ... end`, think: **"the code in the block runs *inside* whatever do_x sets up."**

---

## 7. 🔎 In freee-payroll

**a) `each` then `map` building hashes**: `app/services/payroll/yearend_adjustments/annual_salary_and_bonus_import/fetch_salary_statement_summary_service.rb:14` and `:19-25`
```ruby
      cps_list.each { |cps| initialize_cps(cps) }
```
```ruby
      cps_list.map do |cps|
        {
          status: cps.status,
          reference_date_name: T.must(cps.company_reference_date_rule).name,
          employees_count: cps.employee_payroll_statements.size,
        }
      end
```
- Line 14: `each` for a **side effect** (prepare each monthly payroll run, "cps"). The return value isn't used.
- Lines 19–25: `map` builds a **new array of hashes**, one summary per payroll run. It's the last expression of `execute`, so it's the method's return value. (Ignore `T.must` for now; it's Sorbet, Lesson 8.)

**b) `each_slice` + `flat_map`**: `app/modules/hr_master/infrastructure/repositories/member_repository.rb:30`
```ruby
              employee_ids.each_slice(DB_BATCH_SIZE).flat_map { |batch_ids| base_scope.where(id: batch_ids).to_a }
```
`DB_BATCH_SIZE` is `1000` (line 8). Take the ids 1000 at a time → for each chunk, query the database for those employees → `flat_map` joins all chunks into **one** flat array of employees.

**c) `index_by(&:id)` and iterating a Hash**: `app/models/concerns/company_reference_date_rule/reimbursed_expense_cps_finder.rb:66-75`
```ruby
      result = {}
      rules = where(id: rule_ids).index_by(&:id)

      rules.each do |rule_id, rule|
        begin
          result[rule_id] = rule.find_or_create_cps_with_upcoming_pay_date_with_limit!(date)
        rescue StandardError
          result[rule_id] = nil
        end
      end
```
`rules` becomes `{ id => rule }`. `each` over a Hash gives two block params `|rule_id, rule|`. Each result goes into `result[rule_id]`. (`begin/rescue` is error handling, Lesson 7.)

**d) `select` with a multi-condition block**: `app/models/employee_payroll_statement.rb:533`
```ruby
    candidates = epss.select { |eps| eps.cps.fixed? && eps.employee.present? }
```
Keep only employee payroll statements whose monthly payroll run is **fixed** (確定) AND whose employee still exists.

**e) `sum(&:amount)` inside a lambda**: `app/models/salary_comparison_item.rb:78`
```ruby
    total_other_allowances: lambda { |eps| eps.user_defined_allowances.sum(&:amount) },
```
A Hash entry whose value is a **lambda**: given an `eps`, add up the `amount` of all its user-defined allowances (手当).

**f) `each_with_object`**: `app/models/salary_comparison_item.rb:95`
```ruby
      VALUE_KEYS.each_with_object({}) { |key, hash| hash[key] = values[key].to_i || 0 }
```
Start with an empty Hash `{}`, and for each key store the value converted to an Integer. (Fun fact for a careful reader: `.to_i` never returns `nil`, so the `|| 0` never does anything. Real code isn't always perfect!)

**g) `map { }.sum` and `all?`**: `app/values/resident_tax_monthly_amount.rb:16-18` and `:21`
```ruby
  def sum
    KEYS.map { |key| public_send(key).to_i }.sum
  end
```
```ruby
    self.class == other.class && KEYS.all? { |key| self.send(key) == other.send(key) }
```
`KEYS` is `%i[jun jul aug ...]` (12 months of resident tax, 住民税). `sum` = get each month's amount and total them. The second line is "two objects are equal if same class AND **all** 12 months match."

**h) `find` with a block**: `app/controllers/api/private/tel_support_requests_controller.rb:8`
```ruby
      accountant_company = current_auth_user.companies.find { |company| company.accountant == true }
```
First company of the logged-in user that is an accountant (会計事務所) company.

**i) `group_by`**: `app/services/attendance/attendance_check/check_revoke_service.rb:79`
```ruby
        approval_request.daily_holiday_assigned_result.special_holidays.group_by(&:special_holiday_setting_id)
```
Group special holidays by which special-holiday setting they belong to.

**j) A lambda in a scope**: `app/models/employee_time_clock.rb:42`
```ruby
  scope :start_date, ->(date) { where('datetime >= ?', date.to_date.beginning_of_day) if date.present? }
```
A named query that takes one argument `date`.

**k) Passing a block to a method that `yield`s**: `app/models/concerns/company/release_flags.rb:11`
```ruby
      DatabaseSelector.connected_to_reader { original_parameter(name) }
```
"Run `original_parameter(name)` while connected to the read-replica database." Inside `lib/database_selector.rb` you can find the matching `yield` (e.g. line 54).

---

## 8. Practice (15 min)

In `irb`, predict then check:

```ruby
1.  [1, 2, 3].map { |n| n + 1 }
2.  [1, 2, 3].each { |n| n + 1 }
3.  [5, 8, 11].select { |n| n > 6 }
4.  [5, 8, 11].find { |n| n > 6 }
5.  [5, 8, 11].find { |n| n > 20 }
6.  %w[a b c].map(&:upcase)
7.  (1..10).each_slice(4).to_a
8.  [1, 2, 3].sum { |n| n * 100 }
9.  %w[x y x z x].tally
10. [2, 4, 6].all?(&:even?)
```

Write, using this data:
```ruby
Employee = Struct.new(:id, :name, :dept, :salary)
staff = [
  Employee.new(1, '給与太郎', 'sales', 300_000),
  Employee.new(2, '労務花子', 'dev', 400_000),
  Employee.new(3, '勤怠次郎', 'dev', 350_000),
]
```
(`Struct.new` quickly makes a simple class with those fields; you'll learn classes in Lesson 6.)

11. An array of all names.
12. The total of all salaries.
13. Only employees in `'dev'`.
14. The first employee earning more than 320,000.
15. A Hash `{ 'sales' => [...], 'dev' => [...] }` grouped by dept.
16. A lookup Hash `{ 1 => employee, 2 => employee, ... }` using `to_h` (plain-Ruby equivalent of Rails' `index_by(&:id)`).
17. Write a method `each_twice(list)` that uses `yield` to call the block **two times for each item**.

*Answers: 1 `[2, 3, 4]` · 2 `[1, 2, 3]` (each returns the original) · 3 `[8, 11]` · 4 `8` · 5 `nil` · 6 `["A", "B", "C"]` · 7 `[[1, 2, 3, 4], [5, 6, 7, 8], [9, 10]]` · 8 `600` · 9 `{"x" => 3, "y" => 1, "z" => 1}` · 10 `true`*

```ruby
staff.map(&:name)                         # 11
staff.sum(&:salary)                       # 12 → 1050000
staff.select { |e| e.dept == 'dev' }      # 13
staff.find { |e| e.salary > 320_000 }     # 14 → 労務花子
staff.group_by(&:dept)                    # 15
staff.to_h { |e| [e.id, e] }              # 16 (in Rails: staff.index_by(&:id))
def each_twice(list)                      # 17
  list.each do |item|
    yield item
    yield item
  end
end
each_twice([1, 2]) { |x| puts x }         # 1 1 2 2
```

---

## 9. Summary

- A **block** is code passed to a method: `{ |x| ... }` (one line) or `do |x| ... end` (multi-line).
- `each` = do something (returns original); `map` = transform into a new array; `select`/`reject` = keep/drop; `find` = first match or nil; `any?`/`all?`/`none?`/`count`/`sum`.
- `&:name` = shorthand for `{ |x| x.name }`.
- Reshape: `group_by` (key → many), `index_by` (Rails; key → one), `to_h`, `each_slice(n)` (batches), `flat_map`, `each_with_object`, `partition`, `tally`, `reduce`.
- `->(x) { }` is a **lambda** (a stored block), used in Rails scopes.
- `yield` runs the caller's block; `X.something { ... }` often means "run my code *inside* X's setup" (transactions, read replica, leak-check off).

➡️ Now take **Test5.md**.
