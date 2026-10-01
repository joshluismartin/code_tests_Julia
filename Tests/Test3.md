# Test 3: Ruby Basics 1 (Values, Variables, Methods)

**Rules:** Don't look at Lesson3.md while answering. You MAY use `irb` only for Part D, after writing your answer on paper first. Then check `Answers3.md`.
**Total:** 38 points

---

## Part A: Multiple choice (1 point each)

**1.** What is `nil`?
- a) The number zero
- b) An empty string
- c) Ruby's value for "nothing / no value"
- d) An error

**2.** What is a *Symbol* such as `:company_id`?
- a) A comment
- b) A fixed name/label value, often used as a key or option
- c) A kind of number
- d) A variable that can be reassigned

**3.** Which name follows Ruby convention for a **class**?
- a) `employee_time_clock`
- b) `EMPLOYEE_TIME_CLOCK`
- c) `EmployeeTimeClock`
- d) `employeeTimeClock`

**4.** A method name ending in `?` usually…
- a) raises an error
- b) returns true or false
- c) modifies the object
- d) is private

**5.** A method name ending in `!` usually…
- a) returns true or false
- b) is dangerous: raises an error on failure or modifies the object
- c) is a comment
- d) can't be called

**6.** In `def double(n)`, what is `n`?
- a) An argument
- b) A parameter
- c) A constant
- d) A symbol

**7.** What does *implicit return* mean?
- a) Methods can't return values
- b) You must always write `return`
- c) The value of the last line of the method is returned automatically
- d) The first line is returned

**8.** What does `# frozen_string_literal: true` at the top of a file do?
- a) Makes the file read-only on disk
- b) Makes all string literals in the file unchangeable
- c) Turns off Sorbet
- d) Deletes all strings

**9.** In freee-payroll, which quote style should you use for a string with no interpolation?
- a) Double quotes `"..."`
- b) Single quotes `'...'`
- c) Backticks
- d) Either; there's no rule

---

## Part B: Predict the result (1 point each)

Write exactly what each expression returns.

**10.** `7 / 2`
**11.** `7.0 / 2`
**12.** `9 % 4`
**13.** `'5'.to_i + 5`
**14.** `5.to_s + '5'`
**15.** `'abc'.to_i`
**16.** `nil.to_i`
**17.** `'給与計算'.length`
**18.** Given `year = 2026`, what does `"#{year}年"` return?
**19.** Given `year = 2026`, what does `'#{year}年'` print with `puts`?

---

## Part C: Fill in the missing code (1 point each blank)

**20.** Make this method return `true` when `hours` is more than 40:
```ruby
def ___________(hours)
  hours > 40
___
```
(The method name should be `too_long` written with the correct Ruby convention for a yes/no method.)

**21.** Complete so that `welcome('田中')` returns `"ようこそ、田中さん"`:
```ruby
def welcome(name)
  ___ようこそ、___________さん___
end
```

**22.** Create a constant named `max employees` (convention!) set to 50:
```ruby
_____________ = 50
```

---

## Part D: Write the code (2 points each)

**23.** Write a method `yen(amount)` that returns a string like `"1000円"` for `yen(1000)`.

**24.** Write a method `half(n)` that returns half of `n` **keeping decimals**, so `half(5)` returns `2.5`.

**25.** Write a method `greeting(name, time = 'morning')` that returns `"Good morning, Taro"` for `greeting('Taro')` and `"Good evening, Taro"` for `greeting('Taro', 'evening')`.

---

## Part E: Answer in a sentence (2 points each)

**26.** What is the difference between `puts x` and `p x`?

**27.** Why do controllers often call `.to_i` on values like `params[:id]`?

---

## Part F: Read the real repo (4 points)

**28.** From `app/models/employee_time_clock.rb:47-49`:
```ruby
  def revised?
    revised_at > Time.utc(1000, 1, 1)
  end
```
a) What does this method return, true/false or something else? How do you know from the name? (1)
b) Where is the `return` keyword? Why isn't it needed? (1)
c) In plain words, when does it return `true`? (1)

**29.** From `app/values/birthdate.rb:30`:
```ruby
    (adjusted_date.strftime('%Y%m%d').to_i - birthdate.strftime('%Y%m%d').to_i) / 10_000
```
If `adjusted_date` is 2026-09-30 and `birthdate` is 1996-10-01, what number does this return? Show the math. (1)
