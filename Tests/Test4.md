# Test 4: Ruby Basics 2 (Decisions and Collections)

**Rules:** Don't look at Lesson4.md while answering. Write answers first; you may check Part D in `irb` afterwards. Then check `Answers4.md`.
**Total:** 44 points

---

## Part A: Multiple choice (1 point each)

**1.** Which values are *falsy* in Ruby?
- a) `nil`, `false`, `0`, `''`
- b) Only `nil` and `false`
- c) Only `false`
- d) `nil`, `false`, `[]`

**2.** What is a *guard clause*?
- a) A security check on the server
- b) An early `return` (or `raise`) at the top of a method for the "nothing to do / invalid" case
- c) A comment explaining a method
- d) A test

**3.** What does *memoization* (`@x ||= expensive_call`) mean?
- a) Writing notes in comments
- b) Remembering a computed result so the slow work is only done once
- c) Deleting a variable
- d) Freezing a variable

**4.** What does `employee&.name` return when `employee` is `nil`?
- a) It crashes with NoMethodError
- b) `''`
- c) `nil`
- d) `false`

**5.** Which is true about `blank?` / `present?`?
- a) They are built into plain Ruby
- b) They come from Rails' ActiveSupport; `0.blank?` is `false`, `''.blank?` is `true`
- c) `[].present?` is `true`
- d) `nil.present?` is `true`

**6.** What is the difference between `1..5` and `1...5`?
- a) None
- b) `..` includes 5; `...` excludes 5
- c) `..` excludes 5; `...` includes 5
- d) `...` is an endless range

**7.** Which is an **array of symbols**?
- a) `%w[a b]`
- b) `%i[a b]`
- c) `{ a: 1 }`
- d) `(a..b)`

**8.** What does `h.fetch(:missing)` do when `:missing` isn't a key?
- a) Returns `nil`
- b) Returns `false`
- c) Raises a `KeyError`
- d) Adds the key

---

## Part B: Predict the result (1 point each)

**9.** `[] ? 'yes' : 'no'`
**10.** `false || 'fallback'`
**11.** `[3, 6, 9][0]`
**12.** `[3, 6, 9][-1]`
**13.** `[3, 6, 9][3]`
**14.** `{ id: 7 }[:id]`
**15.** `{ id: 7 }['id']`
**16.** `(1..4).to_a`
**17.** `[1, nil, 2, nil].compact`
**18.** `x = 'a'; x ||= 'b'; x`
**19.** (Rails) `'   '.blank?`
**20.** (Rails) `''.presence`

---

## Part C: Fill in the missing code (1 point each blank)

**21.** Leave the method early when `company` is nil:
```ruby
def company_name(company)
  ______ nil __ company.nil?
  company.name
end
```

**22.** Return `'adult'` if age is 18 or more, otherwise `'minor'`, using the ternary operator:
```ruby
def category(age)
  age >= 18 _ 'adult' _ 'minor'
end
```

**23.** Memoize the result of `load_settings` in `@settings`:
```ruby
def settings
  @settings ___ load_settings
end
```

**24.** (4 points) Complete the `case` so 1–3 hours gives `'short'`, 4–8 gives `'normal'`, and anything else gives `'long'`:
```ruby
def shift_length(hours)
  ____ hours
  when 1__3 then 'short'
  when 4__8 then 'normal'
  ____ 'long'
  end
end
```

---

## Part D: Write the code (2 points each)

**25.** Write `pay_label(type)` returning `'月給'` for `'monthly'`, `'日給'` for `'daily'`, `'時給'` for `'hourly'`, and `'不明'` for anything else. Use `case/when`.

**26.** Write `safe_length(text)` that returns the length of `text`, or `0` when `text` is `nil`. (Hint: `&.` and `||`.)

**27.** Create a hash `rates` mapping the symbols `:kyokai` → `'協会けんぽ'` and `:kumiai` → `'組合健保'`, freeze it, and write the expression that gets the value for `:kumiai`.

---

## Part E: Answer in a sentence (2 points each)

**28.** Why does freee-payroll code use `.blank?` instead of just `if name` to check "is the name empty"?

**29.** What is the difference between the keys in `{ id: 1 }` and `{ 'id' => 1 }`, and why does it matter?

---

## Part F: Read the real repo (5 points)

**30.** `app/models/user_employee.rb:30`
```ruby
    return if employee&.member_id.blank?
```
Describe the **two** different situations in which this line returns early. (2)

**31.** `app/models/concerns/work_record_summary_validations.rb:5-14`
```ruby
  HOURS_IN_MONTH = 31 * 24
  MINUTES_IN_MONTH = HOURS_IN_MONTH * 60
  MINUTES_IN_MONTH_MESSAGE = "は0から#{HOURS_IN_MONTH}時間までの値を入力してください。"
  MINUTES_IN_MONTH_VALIDATION = {
    numericality: {
      only_integer: true,
      in: 0..MINUTES_IN_MONTH,
      message: MINUTES_IN_MONTH_MESSAGE,
    },
  }.freeze
```
a) What is the value of `MINUTES_IN_MONTH`? (1)
b) Why is `MINUTES_IN_MONTH_MESSAGE` written with double quotes? (1)
c) What does `MINUTES_IN_MONTH_VALIDATION.dig(:numericality, :in)` return? (1)
