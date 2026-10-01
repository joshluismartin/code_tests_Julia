# Lesson 3: Ruby Basics 1 (Values, Variables, Methods)

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Read and write the smallest pieces of Ruby: values, variables, constants, strings, and methods. Also recognize the "magic comments" at the top of freee-payroll files.
**You need:** A terminal. Type `irb` to open Ruby's interactive console (type `exit` to leave). Or run one line with `ruby -e 'puts 1 + 1'`.

---

## 1. What is Ruby?

| Term | Meaning in plain words |
|---|---|
| **Ruby** | The programming language of freee-payroll's backend. Designed to read almost like English. |
| **irb** | "Interactive Ruby": type a line of Ruby, press Enter, see the result immediately. Perfect for experiments. |
| **Expression** | Any piece of code that produces a value. `1 + 2` is an expression whose value is `3`. |
| **Statement / line** | One instruction. Ruby runs lines from top to bottom. |
| **Syntax** | The grammar rules of the language. A "syntax error" means Ruby couldn't understand your text. |
| **Object** | In Ruby, *every* value is an object: a thing that has data and can do things (**methods**). Even `5` and `nil`. |
| **Method** | A named action an object can do. `"abc".upcase`: `upcase` is a method of the string. |
| **Return value** | What a method or expression gives back. |

```ruby
# In irb:
1 + 2           # => 3
"abc".upcase    # => "ABC"
```
`# =>` in examples means "the result is".

---

## 2. Basic value types

| Type (class) | Examples | Notes |
|---|---|---|
| **Integer** | `1`, `-5`, `10_000` | Whole numbers. `_` is just for readability: `10_000` = 10000. The repo uses it a lot: `1_040_000`. |
| **Float** | `1.5`, `3.0` | Numbers with a decimal point. (Money in payroll is normally Integer yen, not Float.) |
| **String** | `'hello'`, `"給与"` | Text. |
| **Symbol** | `:company_id`, `:ok` | A fixed name/label. Cheap and used as keys and options everywhere in Rails. |
| **nil** | `nil` | "Nothing / no value". Like an empty box. |
| **Boolean** | `true`, `false` | Yes/no. (Their classes are `TrueClass`/`FalseClass`.) |

Ask any value what it is with `.class`:

```ruby
'a'.class    # => String
:a.class     # => Symbol
nil.class    # => NilClass
1.class      # => Integer
1.5.class    # => Float
true.class   # => TrueClass
```

### 2.1 Numbers and math

```ruby
1 + 2 * 3   # => 7      (* before +, like school math)
2**10       # => 1024   (power)
7 / 2       # => 3      ⚠️ Integer ÷ Integer = Integer (the decimal part is dropped)
7.0 / 2     # => 3.5    (use a Float to keep decimals)
7 % 2       # => 1      (% = remainder, "modulo")
```

> ⚠️ **Integer division** drops the remainder. freee-payroll uses this on purpose; see the age calculation in §7.

### 2.2 Converting between types

| Method | Meaning | Example |
|---|---|---|
| `.to_s` | to **s**tring | `5.to_s # => "5"`, `nil.to_s # => ""` |
| `.to_i` | to **i**nteger | `'12'.to_i # => 12`, `'12abc'.to_i # => 12`, `'abc'.to_i # => 0`, `nil.to_i # => 0` |
| `.to_sym` | to symbol | `'name'.to_sym # => :name` |
| `.to_f` | to float | `'1.5'.to_f # => 1.5` |

```ruby
'5'.to_i + 1   # => 6
5.to_s + '1'   # => "51"   (+ on strings joins them)
```

You'll see `.to_i` everywhere in controllers because data from a URL/form always arrives as **text**. For example `show_params[:id].to_i` in `app/controllers/api/private/employee_display_names_controller.rb:11`.

---

## 3. Variables and constants

| Term | Meaning |
|---|---|
| **Variable** | A name that points at a value, so you can use it later. Created with `=` (called **assignment**). |
| **Constant** | A name whose value should not change. Starts with a **Capital letter**; by convention ALL_CAPS. |
| **snake_case** | lowercase words joined by `_`. Used for variables and method names: `employee_id`. |
| **CamelCase** (PascalCase) | Each word capitalized, no spaces. Used for class/module names: `EmployeeTimeClock`. |
| **SCREAMING_SNAKE_CASE** | For constants: `DEFAULT_DELETED_AT`. |

```ruby
employee_name = '給与太郎'   # variable
age = 30
age = age + 1               # now 31 (you can reassign variables)
age += 1                    # shortcut for age = age + 1 → 32

MAX_EMPLOYEES = 50          # constant
```

Reassigning a constant makes Ruby print a warning. Don't do it.

### 3.1 Comments

```ruby
# This is a comment. Ruby ignores everything after # on a line.
total = 100 # comments can follow code too
```

freee-payroll comments are often in Japanese. They usually explain **why**, which is the most valuable information.

### 3.2 Printing

| Method | Prints | Use |
|---|---|---|
| `puts x` | `x` as plain text + a new line | Human-friendly output |
| `p x` | `x` in "inspect" form (shows quotes on strings, `nil` visibly) and **returns** `x` | Debugging |

```ruby
puts 'hi'   # hi
p 'hi'      # "hi"
puts nil    # (blank line)
p nil       # nil
```

`puts` itself returns `nil`. (Try `p(puts('hi'))`: it prints `hi` then `nil`.)

> In freee-payroll app code you **don't** leave `puts`/`p` lying around, and you don't use `Rails.logger` either (the `Hr/UsedRailsLogger` RuboCop cop forbids it; see `app/README.md`). They're for experiments in irb and debugging only.

---

## 4. Strings

### 4.1 Single vs double quotes

```ruby
name = 'Taro'
puts "Hi #{name}"   # Hi Taro        ← double quotes: #{...} is replaced (interpolation)
puts 'Hi #{name}'   # Hi #{name}     ← single quotes: printed literally
```

- **String interpolation** = inserting a value inside a string with `#{ }`. It only works in **double quotes**.
- **freee-payroll rule:** use **single quotes** unless you need interpolation (`.rubocop.yml`: `Style/StringLiterals: EnforcedStyle: single_quotes`). So when you see double quotes in the repo, look for a `#{}` inside.

### 4.2 Useful string methods

```ruby
'abc'.upcase          # => "ABC"
'ABC'.downcase        # => "abc"
'給与'.length          # => 2     (counts characters, even Japanese)
'ab' * 3              # => "ababab"
'  hi  '.strip        # => "hi"
'a,b,c'.split(',')    # => ["a", "b", "c"]
'hello'.include?('ell') # => true
'hello'.start_with?('he') # => true
```

### 4.3 Frozen strings

`.freeze` makes an object **unchangeable** (immutable). Trying to modify it raises a `FrozenError`.

```ruby
s = 'x'.freeze
s.frozen?   # => true
s << 'y'    # FrozenError (can't modify frozen String)
```

---

## 5. Methods (defining your own)

A **method** is a named, reusable piece of code. **Define** it with `def ... end`, then **call** it by name.

```ruby
def double(n)      # n is a PARAMETER (an input placeholder)
  n * 2
end

double(4)          # => 8     4 is the ARGUMENT (the actual value passed in)
double 4           # => 8     parentheses are optional when calling
```

| Term | Meaning |
|---|---|
| **Define** | Write the method (`def`). |
| **Call / invoke** | Run it (`double(4)`). |
| **Parameter** | The name inside the definition (`n`). |
| **Argument** | The value you pass when calling (`4`). |
| **Implicit return** | Ruby automatically returns the value of the **last line**. No `return` keyword needed. |
| **Early return** | `return x` exits the method immediately (used for "guard clauses", Lesson 4). |
| **Default argument** | A parameter with a fallback value if the caller doesn't pass one. |

```ruby
def greet(name, greeting = 'Hello')
  "#{greeting}, #{name}!"
end

greet('Taro')        # => "Hello, Taro!"
greet('Taro', 'Hi')  # => "Hi, Taro!"
```

One-line ("endless") method style, available in Ruby 3:
```ruby
def adult?(age) = age >= 18
adult?(20)  # => true
```

### 5.1 `?` and `!` at the end of method names

These are **naming conventions** (just part of the name, not operators):

| Ending | Convention | Examples |
|---|---|---|
| `?` | Returns true/false (a **predicate**) | `revised?`, `empty?`, `current_company.xxx_enabled?` |
| `!` | "Be careful": either **raises an error** on failure, or **modifies** the object in place | `save!`, `update!`, `soft_delete!`, `validate!` |

Example: in Rails, `save` returns `false` if it fails, but `save!` **raises an error**. You'll use this in Lesson 11.

### 5.2 Calling methods on objects

```ruby
'abc'.upcase         # call upcase on a string
employee.company_id  # call company_id on an employee object
```
The object before the `.` is called the **receiver**. Chains like `a.b.c` run left to right: call `b` on `a`, then `c` on that result.

---

## 6. Magic comments at the top of files

Open almost any freee-payroll Ruby file and you'll see one or two special comments at the top:

```ruby
# typed: strict
# frozen_string_literal: true
```

| Comment | Meaning |
|---|---|
| `# typed: false / true / strict` | A **Sorbet sigil**: how strictly the type checker checks this file (Lesson 8). In `app/`, `typed: true` is by far the most common (~4,100 files), then `strict` (~190) and `false` (~115). Many files have no sigil at all. |
| `# frozen_string_literal: true` | All string literals written in this file are automatically frozen (like calling `.freeze` on each). Saves memory and prevents accidental changes. Only a minority of freee-payroll files have it (~700 of ~5,600 in `app/`). That's why older files write `.freeze` explicitly. |

---

## 7. 🔎 In freee-payroll

Open each file and find the line.

**a) A constant**: `config/constants.rb:51`
```ruby
DEFAULT_DELETED_AT = Time.utc(1000, 1, 1)
```
A constant holding the time "year 1000, Jan 1, UTC". `Time.utc(...)` calls the method `utc` on the class `Time`. freee-payroll uses this date to mean "**not deleted**" (Lesson 13).

**b) Frozen string constants**: `app/models/slack_integration.rb:3-7`
```ruby
  ENCRYPTED_TOKEN_KEY = 'encrypted_token'.freeze
  ENCRYPTED_TOKEN_IV_KEY = 'encrypted_token_iv'.freeze
  BOT_USER_ID = 'bot_user_id'.freeze
  TEAM_ID = 'team_id'.freeze
  TEAM_NAME = 'team_name'.freeze
```
Single quotes (no interpolation needed) + `.freeze` (this file doesn't have the `frozen_string_literal` comment, so it freezes by hand).

**c) Predicate methods with implicit return**: `app/models/employee_time_clock.rb:47-49` and `59-61`
```ruby
  def revised?
    revised_at > Time.utc(1000, 1, 1)
  end
```
```ruby
  def modified?
    datetime != original_datetime
  end
```
- `?` means they return true/false.
- No `return`: the comparison on the last line *is* the return value.
- `!=` means "not equal". `revised?` is true when `revised_at` is later than the "empty" date 1000-01-01.

**d) String interpolation (double quotes because of `#{}`)**: `app/models/employee_time_clock.rb:69`
```ruby
      self.errors.add(:base, "#{EmployeeTimeClock.human_enum_name(:type, type)}日時は出勤日かその翌日のみ指定できます。")
```
The part inside `#{ }` produces a word like "退勤" (clock-out), and the message becomes "退勤日時は出勤日かその翌日のみ指定できます。" ("Clock-out time can only be on the work day or the next day."). `:base` is a Symbol.

**e) Integer division on purpose**: `app/values/birthdate.rb:25-31`
```ruby
  def age_at(date:, based_on_nenrei_keisan_ni_kansuru_horitsu: false)
    adjusted_date = based_on_nenrei_keisan_ni_kansuru_horitsu ? date + 1.day : date

    raise ArgumentError, '生年月日よりも前の日付が指定されています。' if adjusted_date < birthdate

    (adjusted_date.strftime('%Y%m%d').to_i - birthdate.strftime('%Y%m%d').to_i) / 10_000
  end
```
Don't worry about every piece yet (`date:` is a *keyword argument*, Lesson 7; `? :` is Lesson 4; `raise` is Lesson 7). Look at the last line:
- `strftime('%Y%m%d')` turns a date into text like `'20250101'`, and `.to_i` turns that into the number `20250101`.
- Birthdate 2000-01-02 → `20000102`. Today 2025-01-01 → `20250101`.
- `(20250101 - 20000102) / 10_000` = `249999 / 10_000` = **24** (Integer division drops `.9999`). The person is 24, not yet 25. A neat trick that relies on integer division!
- It's the last line, so it's the return value.

---

## 8. Practice (15 min)

Open `irb` and type each. **Predict the result first**, then check.

```ruby
1. 10 / 3
2. 10 % 3
3. 10.0 / 4
4. '3'.to_i * 2
5. 3.to_s * 2
6. nil.to_s.length
7. name = 'Hanako'; "Hello, #{name}"
8. 'Hello, #{name}'
9. :employee_id.class
10. 1_000_000 + 1
```

Then write these methods in irb:

11. `def tax(amount)` that returns 10% of `amount` as an Integer (hint: `amount * 10 / 100`).
12. `def full_name(last, first)` that returns `"#{last} #{first}"`. `full_name('給与', '太郎')` should give `"給与 太郎"`.
13. `def overtime?(hours)` that returns true when hours > 8.
14. `def label(name, suffix = 'さん')` so that `label('田中')` → `"田中さん"` and `label('田中', '様')` → `"田中様"`.

*Answers: 1 → `3` · 2 → `1` · 3 → `2.5` · 4 → `6` · 5 → `"33"` · 6 → `0` · 7 → `"Hello, Hanako"` · 8 → `"Hello, \#{name}"` (irb shows the `#` escaped as `\#`) · 9 → `Symbol` · 10 → `1000001`.*
*11 `def tax(amount) = amount * 10 / 100` · 12 `def full_name(last, first) = "#{last} #{first}"` · 13 `def overtime?(hours) = hours > 8` · 14 `def label(name, suffix = 'さん') = "#{name}#{suffix}"`*

---

## 9. Summary

- Every value is an **object**; `.class` tells you its type: Integer, Float, String, Symbol, nil, true/false.
- `Integer / Integer` drops decimals; `%` is remainder; `10_000` is just 10000.
- Variables are `snake_case`; constants are `CAPITALIZED`; classes are `CamelCase`.
- Use **single quotes** in freee-payroll unless you need `#{interpolation}` (double quotes).
- `def name(params) ... end`; the **last line is returned** automatically; `?` = returns boolean, `!` = raises/modifies.
- `.freeze` / `# frozen_string_literal: true` make strings immutable; `# typed:` is the Sorbet level.

➡️ Now take **Test3.md**.
