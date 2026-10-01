# Test 5: Ruby Basics 3 (Blocks and Iteration)

**Rules:** Don't look at Lesson5.md while answering. Write your answers first; you may check Parts B and D in `irb` afterwards. Then check `Answers5.md`.
**Total:** 44 points

---

## Part A: Multiple choice (1 point each)

**1.** What is a *block*?
- a) A kind of variable
- b) A chunk of code passed to a method so the method can run it
- c) A database table
- d) A comment

**2.** What does `map` return?
- a) The original array
- b) A new array containing the block's result for each item
- c) The first matching item
- d) `true` or `false`

**3.** What does `each` return?
- a) A new array of results
- b) The original collection (it's used for side effects)
- c) `nil` always
- d) The last item

**4.** Which method returns the **first** item that matches, or `nil`?
- a) `select`
- b) `find`
- c) `any?`
- d) `map`

**5.** `employees.map(&:id)` is the same as:
- a) `employees.map { |e| e.id }`
- b) `employees.select { |e| e.id }`
- c) `employees.id`
- d) `employees.map { |id| id }`

**6.** What does Rails' `index_by(&:id)` build?
- a) An array of ids
- b) A Hash `{ id => item }` for fast lookup
- c) A Hash `{ id => [items] }`
- d) A sorted array

**7.** What is the main use of `each_slice(1000)` in freee-payroll?
- a) Sorting
- b) Processing large lists in batches of 1000 to protect memory / query limits
- c) Removing duplicates
- d) Printing

**8.** What is `->(x) { x * 2 }`?
- a) A hash
- b) A range
- c) A lambda (a block stored as a value)
- d) A syntax error

**9.** Inside a method, what does `yield` do?
- a) Stops the program
- b) Runs the block the caller passed in
- c) Returns `nil`
- d) Creates a variable

---

## Part B: Predict the result (1 point each)

**10.** `[1, 2, 3].map { |n| n * 3 }`
**11.** `[1, 2, 3].each { |n| n * 3 }`
**12.** `[4, 7, 10].select { |n| n.even? }`
**13.** `[4, 7, 10].reject(&:even?)`
**14.** `[4, 7, 10].find { |n| n > 5 }`
**15.** `[4, 7, 10].any? { |n| n > 9 }`
**16.** `[4, 7, 10].sum`
**17.** `(1..5).each_slice(2).to_a`
**18.** `%w[a b a].tally`
**19.** `[[1], [2, 3]].flat_map { |a| a }`
**20.** `{ x: 1, y: 2 }.map { |k, v| v * 10 }`

---

## Part C: Fill in the missing code (1 point each blank)

**21.** Get all employee ids using the shorthand:
```ruby
employee_ids = employees.___(__:id)
```

**22.** Keep only fixed payroll statements:
```ruby
fixed = statements.______ { |s| s.fixed? }
```

**23.** Group records by `employee_id`:
```ruby
by_employee = records.________(&:employee_id)
```

**24.** Complete the method so it runs the caller's block and returns the block's value, printing `'begin'` before and `'finish'` after:
```ruby
def around
  puts 'begin'
  result = _____
  puts 'finish'
  ______
end
```

---

## Part D: Write the code (2 points each)

Use this data:
```ruby
Row = Struct.new(:employee_id, :amount)
rows = [Row.new(1, 1000), Row.new(2, 500), Row.new(1, 300)]
```

**25.** The total of all `amount`s.

**26.** An array of the unique employee ids (`[1, 2]`).

**27.** A Hash `{ employee_id => total amount for that employee }`, i.e. `{ 1 => 1300, 2 => 500 }`. (Hint: `group_by` then `transform_values`, or `each_with_object(Hash.new(0))`.)

**28.** `true` if any row's amount is greater than 900.

---

## Part E: Answer in a sentence (2 points each)

**29.** When should you use `each` and when `map`?

**30.** Why build a lookup hash with `index_by(&:id)` instead of calling `find { |e| e.id == x }` inside a loop?

---

## Part F: Read the real repo (6 points)

**31.** `app/modules/hr_master/infrastructure/repositories/member_repository.rb:30` (`DB_BATCH_SIZE` is 1000)
```ruby
              employee_ids.each_slice(DB_BATCH_SIZE).flat_map { |batch_ids| base_scope.where(id: batch_ids).to_a }
```
If `employee_ids` has 2,500 ids: how many database queries run, and what shape is the final result? (2)

**32.** `app/models/concerns/company_reference_date_rule/reimbursed_expense_cps_finder.rb:67-69`
```ruby
      rules = where(id: rule_ids).index_by(&:id)

      rules.each do |rule_id, rule|
```
What kind of value is `rules`, and what are `rule_id` and `rule` inside the block? (2)

**33.** `app/values/resident_tax_monthly_amount.rb:16-18` (`KEYS` = the 12 months `%i[jun jul ... may]`)
```ruby
  def sum
    KEYS.map { |key| public_send(key).to_i }.sum
  end
```
Explain in words what this method returns. (2)
