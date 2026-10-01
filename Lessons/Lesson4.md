# Lesson 4: Ruby Basics 2 (Decisions and Collections)

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Read and write code that *chooses* what to do (`if`, `unless`, `case`, `? :`, `&.`, `||=`) and code that holds *many* values (Arrays, Hashes, Ranges).
**You need:** `irb` (type `irb` in a terminal).

---

## 1. Comparisons and booleans

A **condition** is an expression that is true or false.

| Operator | Meaning | Example → result |
|---|---|---|
| `==` | equal | `1 == 1` → `true` |
| `!=` | not equal | `'a' != 'b'` → `true` |
| `>` `<` `>=` `<=` | greater / less (or equal) | `5 >= 5` → `true` |
| `&&` | AND: both must be true | `true && false` → `false` |
| `\|\|` | OR: at least one true | `false \|\| true` → `true` |
| `!` | NOT: flip it | `!true` → `false` |

> ⚠️ `=` **assigns** a value; `==` **compares**. `if x = 5` is a classic bug.

### 1.1 Truthiness: Ruby's most important rule

In a condition, **only `nil` and `false` count as false ("falsy")**. *Everything else* is true ("truthy"), including `0`, `''` (empty string) and `[]` (empty array).

```ruby
0   ? 'truthy' : 'falsy'   # => "truthy"   ⚠️ different from JavaScript/PHP!
''  ? 'truthy' : 'falsy'   # => "truthy"
[]  ? 'truthy' : 'falsy'   # => "truthy"
nil ? 'truthy' : 'falsy'   # => "falsy"
```

Because `''` and `[]` are truthy, Rails adds **`blank?` / `present?`** (next section).

### 1.2 `&&` and `||` return a value, not just true/false

```ruby
nil || 'x'      # => "x"     (first truthy value)
false || nil    # => nil
true && nil     # => nil     (&& returns the last value if all truthy, else the first falsy)
```
That's why `name || 'no name'` is a common way to write a **fallback**.

---

## 2. Rails helpers: `blank?`, `present?`, `presence`

These come from **ActiveSupport**, a library that ships with Rails. **They don't exist in plain Ruby/irb.** You can use them in the Rails console (`bin/rails console`) or anywhere in freee-payroll.

| Value | `.blank?` | `.present?` |
|---|---|---|
| `nil` | true | false |
| `false` | true | false |
| `''` (empty string) | true | false |
| `'   '` (only spaces) | true | false |
| `[]`, `{}` (empty) | true | false |
| `0` | **false** | **true** |
| `'abc'`, `[1]` | false | true |

- `present?` is exactly `!blank?`.
- `x.presence` returns `x` if it's present, otherwise `nil`. Example: `''.presence` → `nil`, `'abc'.presence` → `'abc'`.

freee-payroll uses these constantly: `.present?`/`.blank?`/`.presence` appear over 8,000 times in `app/`.

---

## 3. Making decisions

### 3.1 `if / elsif / else`

```ruby
def pay_type_label(type)
  if type == 'monthly'
    '月給'
  elsif type == 'daily'
    '日給'
  else
    '時給'
  end
end
```
`if` is also an expression; it returns the value of the branch that ran, so the method returns the label.

### 3.2 `unless` = "if not"

```ruby
unless user.admin?
  raise 'not allowed'
end
```
Rule of thumb: use `unless` only with a simple condition, and never with `else` (hard to read).

### 3.3 Modifier form ("guard clause")

Put `if`/`unless` **after** a single statement:

```ruby
return if employee.nil?          # leave the method early when there's no employee
raise 'too big' if amount > 100
```

A **guard clause** = an early `return` at the top of a method that handles the "nothing to do" case, so the rest of the method stays simple. You'll see this *everywhere* in freee-payroll.

```ruby
def status(n)
  return 'none' if n.nil?
  n > 0 ? 'positive' : 'zero or negative'
end
status(nil)  # => "none"
status(3)    # => "positive"
status(0)    # => "zero or negative"
```

### 3.4 Ternary operator `condition ? a : b`

A one-line if/else: "if condition then a, else b".

```ruby
label = age >= 18 ? 'adult' : 'minor'
```

### 3.5 `case / when`

Compare one value against many options:

```ruby
access = case role
         when :admin, :owner then 'full'     # several matches separated by commas
         when :staff then 'limited'
         else 'none'
         end
```

`when` can take a **Range** (see §6), which is perfect for tax brackets:

```ruby
case 1_500_000
when -Float::INFINITY..1_320_000 then 'A'
when 1_320_001..3_360_000 then 'B'   # ← this one matches
else 'C'
end
# => "B"
```

---

## 4. Handling `nil` safely: `&.` and `||=`

### 4.1 Safe navigation `&.`

Calling a method on `nil` crashes: `nil.length` → `NoMethodError`. `&.` means "call this method **only if the thing isn't nil**; otherwise just give `nil`."

```ruby
x = nil
x&.length       # => nil  (no crash)
'abc'&.length   # => 3
```

`employee&.member_id` reads as "the employee's member_id, or nil if there's no employee."

### 4.2 `||=` memoization

`a ||= b` means "if `a` is nil or false, set it to `b`; otherwise keep it."

```ruby
def calc
  puts 'calculating'
  42
end

@c ||= calc   # prints "calculating", @c = 42
@c ||= calc   # prints nothing; @c is already set, so calc is not called again
```

**Memoization** = remembering the result of slow work (like a database query) so you only do it once. (`@c` is an *instance variable*, Lesson 6.)

---

## 5. Arrays: ordered lists

An **Array** is an ordered list of values, written with `[ ]`. Positions (**indexes**) start at **0**.

```ruby
a = [10, 20, 30]
a[0]            # => 10      first element
a[-1]           # => 30      negative index counts from the end
a[5]            # => nil     out of range → nil (no crash)
a.length        # => 3
a.first         # => 10
a.last          # => 30
a.include?(20)  # => true
a << 40         # add to the end → [10, 20, 30, 40]
a.sum           # => 100
a.empty?        # => false
[].empty?       # => true
```

More built-ins: `[3, 1, 2].sort` → `[1, 2, 3]`, `[1, 2, 2, 3].uniq` → `[1, 2, 3]`, `[nil, 1, nil].compact` → `[1]` (removes nils), `[[1, 2], [3]].flatten` → `[1, 2, 3]`.

### Shortcut literals

```ruby
%w[general payroll bonus]   # => ["general", "payroll", "bonus"]   (array of Words = strings)
%i[dialog banner]           # => [:dialog, :banner]                (array of symbols)
```
RuboCop in freee-payroll prefers these shortcuts, so you will see them a lot.

---

## 6. Hashes: key → value lookups

A **Hash** stores **key → value** pairs (like a dictionary, or a JSON object). Written with `{ }`.

```ruby
h = { id: 1, name: '給与太郎' }    # keys are the symbols :id and :name
h[:id]            # => 1
h[:missing]       # => nil          (missing key → nil)
h.keys            # => [:id, :name]
h.values          # => [1, "給与太郎"]
h.key?(:name)     # => true
h[:age] = 30      # add/update a key
h.fetch(:name)    # => "給与太郎"
h.fetch(:zzz)     # KeyError!  fetch crashes on missing keys (useful when the key MUST exist)
```

Two ways to write keys:

```ruby
{ id: 1 }          # symbol key, modern style (used in freee-payroll). Same as { :id => 1 }
{ 'id' => 1 }      # string key: access with h['id']
```
⚠️ `{ id: 1 }[:id]` works but `{ id: 1 }['id']` is `nil`: symbol and string keys are **different**. JSON parsed from a request usually has **string** keys unless converted.

(How irb *prints* a hash depends on the Ruby version: Ruby 3.4 shows `{id: 1}`, Ruby 3.3, which the repo uses, shows `{:id=>1}`. Same thing.)

**Nested hashes** and `dig`:

```ruby
setting = { numericality: { only_integer: true } }
setting[:numericality][:only_integer]  # => true
setting.dig(:numericality, :only_integer)  # => true   (dig = safe multi-level lookup)
{ a: nil }.dig(:a, :b)                  # => nil, no crash
```

**Hash as the last argument**: when a hash is the last argument, you can drop the braces:
```ruby
validates :name, presence: true, length: { maximum: 255 }
# is the same as
validates(:name, { presence: true, length: { maximum: 255 } })
```
This is why Rails code looks like English sentences.

---

## 7. Ranges

A **Range** is a span of values.

| Code | Means | `.to_a` |
|---|---|---|
| `1..5` | 1 to 5, **including** 5 | `[1, 2, 3, 4, 5]` |
| `1...5` | 1 to 5, **excluding** 5 | `[1, 2, 3, 4]` |
| `date..` | from `date` onward (no end, "endless range") | n/a |

```ruby
(1..12).include?(12)    # => true
(1...12).include?(12)   # => false
```
Used for validation limits (`in: 0..MINUTES_IN_MONTH`) and date queries (`where(date: start_date..end_date)`).

---

## 8. 🔎 In freee-payroll

**a) Guard clause + `&.` + `blank?`**: `app/models/user_employee.rb:29-30`
```ruby
  def clear_bumon_manager_relations
    return if employee&.member_id.blank?
```
Read it right to left: "take the employee (or nil if none) → its `member_id` → if that's blank (nil or empty) → **leave the method now**." Note `nil.blank?` is `true`, so if there's no employee the `&.` gives `nil`, then `nil.blank?` is true, and we return. One line handles two "nothing to do" cases.

**b) Memoization with `||=`**: `app/models/slack_integration.rb:65-67`
```ruby
  def bot_user_id_record
    @bot_user_id_record ||= integration_parameters.find_or_create_by(company_id: company.id, key: BOT_USER_ID)
  end
```
The first call runs a database query; later calls reuse `@bot_user_id_record`. Also notice the **hash arguments without braces**: `company_id: company.id, key: BOT_USER_ID`.

**c) A frozen Hash constant + array shortcut**: `app/models/banner.rb:4-11` and `:16`
```ruby
  BANNER_TYPES = {
    error: 'エラー',
    warning: '警告',
    info: '情報',
    success: '成功',
    assistance: 'アシスタンス',
    discovery: 'リリース',
  }.freeze
```
```ruby
  validates :banner_type, presence: true, inclusion: { in: BANNER_TYPES.keys.map(&:to_s) }
```
`BANNER_TYPES.keys` → `[:error, :warning, ...]`; `.map(&:to_s)` (Lesson 5) turns them into strings. So a banner's type must be one of `'error'`, `'warning'`, and so on. Note the trailing comma after the last pair; the repo's formatter adds it.

**d) `%w` array**: `app/models/concerns/company_rules/bank_account_validations.rb:19`
```ruby
    validates :type_code, allow_blank: true, inclusion: %w[general payroll bonus]
```

**e) Range inside a nested Hash**: `app/models/concerns/work_record_summary_validations.rb:5-14`
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
Constants built from other constants (744 hours, 44,640 minutes), a double-quoted string because of `#{}`, and a Range `0..44640`: "work minutes must be a whole number between 0 and 44,640."

**f) `case/when` with Ranges (a tax table!)**: `app/values/payroll/v2026/nenmatsu_chosei/kiso_kojo_gaku.rb:21-40`
```ruby
          case gokei_shotoku_kingaku
          when -Float::INFINITY..1_320_000
            1_040_000
          when 1_320_001..3_360_000
            1_040_000
          ...
          when 24_500_001..25_000_000
            160_000
          else
            0
          end
```
基礎控除額 (basic deduction) for the 2026 year-end adjustment: given 合計所得金額 (total income), pick the deduction by bracket. `-Float::INFINITY` means "no lower limit". Real Japanese tax law, written as a `case`!

**g) `unless` block + early `return`**: `app/controllers/api/private/attendance_support_banner_controller.rb:5-12`
```ruby
  def show
    unless Omega::FeatureFlag.active?(:attendance_navigation_v1, namespace: :payroll)
      head :not_implemented
      return
    end

    render json: { show_attendance_support_banner: show_attendance_support_banner? }
  end
```
"If the feature flag is NOT on, answer '501 Not Implemented' and stop. Otherwise return JSON." `{ show_attendance_support_banner: ... }` is a Hash that becomes the JSON body.

**h) Ternary**: `app/values/birthdate.rb:26`
```ruby
    adjusted_date = based_on_nenrei_keisan_ni_kansuru_horitsu ? date + 1.day : date
```
"If we must follow the 年齢計算ニ関スル法律 (Age Calculation Act), use the next day; otherwise use the date as is."

---

## 9. Practice (15 min)

In `irb`, predict then check:

```ruby
1.  0 ? 'yes' : 'no'
2.  nil || 'default'
3.  [5, 6, 7][1]
4.  [5, 6, 7][-1]
5.  { a: 1 }[:b]
6.  { 'a' => 1 }[:a]
7.  (1...3).to_a
8.  %i[a b]
9.  nil&.upcase
10. x = nil; x ||= 5; x ||= 9; x
```

Write:

11. A method `bracket(income)` using `case/when` with ranges: `0..999_999` → `'low'`, `1_000_000..4_999_999` → `'mid'`, anything else → `'high'`.
12. A method `display_name(name)` that returns `'(未設定)'` when `name` is `nil`, otherwise `name`. Use `||`.
13. A method `first_word(text)` that returns `nil` if `text` is `nil` (use a guard clause), else `text.split(' ').first`.
14. A hash `employee` with keys `:id` (1), `:name` ('労務花子'), `:department` (`nil`). Then print the name and check `employee.key?(:department)`.

*Answers: 1 `"yes"` · 2 `"default"` · 3 `6` · 4 `7` · 5 `nil` · 6 `nil` (string key ≠ symbol key) · 7 `[1, 2]` · 8 `[:a, :b]` · 9 `nil` · 10 `5`*

```ruby
# 11
def bracket(income)
  case income
  when 0..999_999 then 'low'
  when 1_000_000..4_999_999 then 'mid'
  else 'high'
  end
end
# 12
def display_name(name) = name || '(未設定)'
# 13
def first_word(text)
  return nil if text.nil?
  text.split(' ').first
end
# 14
employee = { id: 1, name: '労務花子', department: nil }
puts employee[:name]
employee.key?(:department) # => true (the key exists even though its value is nil)
```

---

## 10. Summary

- Only **`nil` and `false`** are falsy. `0`, `''`, `[]` are truthy, so use Rails' `blank?` / `present?` / `presence`.
- `if/elsif/else`, `unless`, modifier `return if ...` (guard clause), ternary `c ? a : b`, `case/when` (works with ranges).
- `&.` = call only if not nil; `||=` = set only if nil/false (memoization); `a || b` = fallback.
- Arrays `[ ]` (index from 0, `-1` = last), `%w[]` strings, `%i[]` symbols.
- Hashes `{ key: value }`; symbol keys ≠ string keys; `fetch` raises on missing; `dig` for nested; braces can be dropped for a last-argument hash.
- Ranges `1..5` (inclusive), `1...5` (exclusive), `date..` (endless).

➡️ Now take **Test4.md**.
