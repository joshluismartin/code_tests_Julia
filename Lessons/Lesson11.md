# Lesson 11 — ActiveRecord 1: Models and Queries

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Read and write basic ActiveRecord code: create, read, update, delete records, and build queries with `where`, `order`, `pluck` and friends, the way freee-payroll does it.
**You need:** Lessons 3–10 (Ruby, Sorbet, SQL). Open `app/models/` and `app/modules/*/infrastructure/repositories/`.

---

## 1. ORM and ActiveRecord

| Term | Meaning |
|---|---|
| **ORM** (Object-Relational Mapping) | A library that lets you work with database rows as **Ruby objects**, instead of writing SQL by hand. |
| **ActiveRecord** | Rails' ORM. |
| **Model** | A Ruby class that represents one table. Each **instance** (object) = one row. |
| **Attribute** | A column's value on an object, e.g. `comment.url`. |
| **Record** | An instance of a model (one row). |
| **Persisted** | Saved in the database (has an `id`). A `new` object that isn't saved yet is *not persisted*. |

| Database world | Ruby world |
|---|---|
| table `comments` | class `Comment` |
| row | object `comment` |
| column `url` | method `comment.url` / `comment.url = '...'` |
| `SELECT` | `Comment.find`, `where`, ... |
| `INSERT` | `create`, `save` |
| `UPDATE` | `update`, `save` |
| `DELETE` | `destroy`, `delete` |

---

## 2. The base class: `ApplicationRecord`

Every model inherits from `ApplicationRecord`, which inherits from `ActiveRecord::Base`.

🔎 `app/models/application_record.rb:1-5`:

```ruby
# 全てのModel classはこのApplicationRecordを継承するようにする
# https://github.com/rails/rails/pull/22567
class ApplicationRecord < ActiveRecord::Base
  include OmegaRails::Model::LeakCheckable
  self.abstract_class = true
```

| Line | Meaning |
|---|---|
| comment | "All model classes must inherit from this ApplicationRecord." |
| `include OmegaRails::Model::LeakCheckable` | Adds freee's **leak check**: a safety net that raises an error if you touch another person's employee data you're not allowed to see (Lesson 13). |
| `self.abstract_class = true` | `ApplicationRecord` itself has **no table**; it's only a parent for real models. |

The file also sets up the primary + replica DB connection (Lesson 10) and a helper `has_accessible_scope` (Lesson 13).

---

## 3. A real, tiny model

🔎 `app/models/comment.rb:1-8`:

```ruby
class Comment < ApplicationRecord
  belongs_to :company

  validates :company_id, presence: true
  validates :user_id, presence: true
  validates :url, presence: true
  validates :comment, length: { maximum: 255 }
end
```

That's all. Notice what's **not** written: the columns. ActiveRecord reads them from the database (`db/schema.rb`, Lesson 10), so `comment.url`, `comment.page_name`, `comment.created_at` etc. just exist.

(`belongs_to` and `validates` are Lesson 12.)

---

## 4. CRUD: Create, Read, Update, Delete

| Term | Meaning |
|---|---|
| **CRUD** | The four basic operations on data: **C**reate, **R**ead, **U**pdate, **D**elete. |

All examples below are **(illustrative example)** console code; the patterns are exactly what the repo uses.

### 4.1 Create

```ruby
comment = Comment.new(company_id: 10, user_id: 5, url: '/home', comment: 'hi')  # in memory only
comment.persisted?   # => false
comment.save         # INSERT; returns true or false
comment.id           # => 123 (now it has an id)

Comment.create(company_id: 10, user_id: 5, url: '/home', comment: 'hi')   # new + save in one step
Comment.create!(company_id: 10, user_id: 5, url: '/home', comment: 'hi')  # raises error if invalid
```

### 4.2 Read

```ruby
Comment.find(123)                 # by id; raises ActiveRecord::RecordNotFound if missing
Comment.find_by(url: '/home')     # first match or nil (no error)
Comment.first / Comment.last      # by id order
Comment.all                       # every row (careful!)
```

| Method | Not found → |
|---|---|
| `find(id)` | raises `ActiveRecord::RecordNotFound` (→ usually a 404) |
| `find_by(...)` | returns `nil` |

### 4.3 Update

```ruby
comment = Comment.find(123)
comment.comment = 'edited'   # change in memory
comment.save                 # UPDATE

comment.update(comment: 'edited again')    # assign + save; true/false
comment.update!(comment: 'edited again')   # raises if invalid
```

### 4.4 Delete

```ruby
comment.destroy     # DELETE the row (runs callbacks, Lesson 12)
comment.destroy!    # same, raises if it fails
```

> ⚠️ **freee-payroll warning:** `destroy`, `destroy!`, `delete`, `delete_all`, `destroy_all` are **physical deletes** (the row is gone). For tables with a `deleted_at` column, deleting usually means a **soft delete**: `update(deleted_at: Time.current)` or the helper `destroy_logically!` (Lesson 13). `app/README.md` warns about this.

---

## 5. Bang methods: `save` vs `save!`

Recall from Ruby: a `!` at the end of a method name means "the dangerous/strict version."

| Non-bang | Bang | Difference |
|---|---|---|
| `save` | `save!` | `save` returns `false` if validation fails; `save!` **raises** `ActiveRecord::RecordInvalid` |
| `create` | `create!` | same idea |
| `update` | `update!` | same idea |
| `destroy` | `destroy!` | `destroy!` raises if it couldn't delete |

**freee-payroll mostly uses bang methods** (`create!`, `update!`, `save!`). Why? If something goes wrong, you *want* an exception: it stops the transaction, gets handled by the controller's error handling (`ErrorHandleable` turns `RecordInvalid` into a friendly 400 message, Lesson 15), and nobody silently ignores a `false`.

🔎 `app/modules/payroll/infrastructure/repositories/company_daily_prorate_setting_repository.rb:21-29`:

```ruby
          def save(company_daily_prorate_setting:)
            company = Company.find(company_daily_prorate_setting.company_id)
            current_setting = company.company_daily_prorate_setting
            if current_setting.nil?
              company.create_company_daily_prorate_setting!(round_type: company_daily_prorate_setting.round_type)
            else
              current_setting.update!(round_type: company_daily_prorate_setting.round_type)
            end
          end
```

"Find the company. If it doesn't have a setting yet, create one (`create_..._!`), otherwise update it (`update!`)."

---

## 6. Queries: building SQL with Ruby

### 6.1 `where`

```ruby
Comment.where(company_id: 10)                      # WHERE company_id = 10
Comment.where(company_id: 10, user_id: 5)          # ... AND user_id = 5
Comment.where(id: [1, 2, 3])                       # WHERE id IN (1, 2, 3)
Comment.where(created_at: Date.new(2026, 1, 1)..)  # WHERE created_at >= '2026-01-01'
Comment.where.not(user_id: 5)                      # WHERE user_id != 5
Comment.where('created_at >= ?', 1.week.ago)       # SQL string with a placeholder
```

| Term | Meaning |
|---|---|
| **Placeholder `?`** | In a SQL string, `?` is replaced safely by the next argument. **Never** put user input into the string with `#{}` because that allows **SQL injection** (an attacker writing SQL through input). |

🔎 Real placeholder use, `app/models/employee_time_clock.rb:42-43`:

```ruby
  scope :start_date, ->(date) { where('datetime >= ?', date.to_date.beginning_of_day) if date.present? }
  scope :end_date, ->(date) { where('datetime <= ?', date.to_date.end_of_day) if date.present? }
```

### 6.2 Other query methods

| Method | SQL idea | Returns |
|---|---|---|
| `order(created_at: :desc)` | `ORDER BY created_at DESC` | relation |
| `limit(10)` | `LIMIT 10` | relation |
| `select(:id, :url)` | `SELECT id, url` | relation (objects with only those columns) |
| `distinct` | `SELECT DISTINCT` | relation |
| `joins(:company)` | `INNER JOIN companies` | relation |
| `count` | `SELECT COUNT(*)` | Integer |
| `exists?` | `SELECT 1 ... LIMIT 1` | true/false |
| `pluck(:id)` | `SELECT id` | **Array** of plain values, e.g. `[1, 2, 3]` |
| `first`, `last`, `find_by` | `LIMIT 1` | one record or nil |
| `find_or_create_by(...)` | find, else insert | record |

### 6.3 Relations are lazy

| Term | Meaning |
|---|---|
| **Relation** (`ActiveRecord::Relation`) | An object that *describes* a query. It is **not** run yet. |
| **Lazy** | The SQL is only sent when you actually need the data: `each`, `map`, `to_a`, `first`, `count`, `pluck`, printing it... |
| **Chaining** | Calling query methods one after another; each returns a new relation. |

```ruby
scope = company.comments                 # no SQL yet
scope = scope.where(user_id: 5)          # still no SQL
scope = scope.order(created_at: :desc)   # still no SQL
scope.to_a                               # NOW: one SQL with all conditions
```

🔎 This is exactly how freee-payroll builds queries step by step, `app/modules/payroll/infrastructure/repositories/company_allowance_repository.rb:45-48`:

```ruby
    def find_all(company_id:, frequencies: nil, is_active: nil)
      company = Company.find(company_id)
      scope = company.company_allowances.where(is_system_created: false)
      scope = scope.where(frequency: frequencies) if frequencies.present?
```

"Start with this company's allowances that aren't system-created; **only if** `frequencies` were given, add another condition."

### 6.4 `pluck`: just the values

🔎 `app/modules/payroll/application/services/update_company_daily_prorate_setting_service.rb:28`:

```ruby
                cps_ids = company.company_payroll_statements.variable.pluck(:id)
```

`pluck(:id)` returns an Array of Integers without building full Ruby objects. That makes it much lighter in memory (freee-payroll's memory rule, Lesson 13).

### 6.5 `exists?`

🔎 `app/modules/hr_portal/application/services/update_employee_hr_portal_sub_action_service.rb:29`:

```ruby
            raise Forbidden unless company.employees.accessible.exists?(id: employee_id)
```

"If this employee isn't among the company's employees *that I'm allowed to access*, raise Forbidden."

---

## 7. freee-payroll rule: query through the company

Look at every repository line so far: they **start from a company**, e.g. `Company.find(company_id).comments...`, not `Comment.where(company_id: ...)`.

🔎 `app/modules/application/infrastructure/repositories/comment_repository.rb:15-18`:

```ruby
          def find_list_by_url(company_id:, url:)
            company = Company.find(company_id)
            comments = url.nil? ? company.comments : company.comments.where(url: url.url)
            comments.map do |comment|
```

Two reasons:

1. **Tenant safety.** `company.comments` automatically adds `WHERE company_id = ...`. You can't accidentally read another company's data.
2. **A RuboCop rule enforces it.** `app/README.md` says: direct query chains on ActiveRecord classes (like `Employee.where(...)`) are violations of the `Freee/QueryFromActiveRecordClass` cop unless the class is on an allowlist. You should go through a **model scope**, an **association** like `company.employees`, or a **Repository**. 🔎 The allowlist is in `.rubocop.yml:125` onward, and `Company` is on it, which is why `Company.find(company_id)` is fine.

| ❌ Avoid | ✅ Prefer |
|---|---|
| `Employee.where(company_id: company.id)` | `company.employees` |
| `Comment.find(id)` | `company.comments.find(id)` |
| `Comment.where(company_id: id, url: url)` | `company.comments.where(url: url)` |

---

## 8. Errors you'll meet

| Error | When |
|---|---|
| `ActiveRecord::RecordNotFound` | `find(id)` didn't find the row (also `company.comments.find(id)` when it belongs to another company!) |
| `ActiveRecord::RecordInvalid` | `save!` / `create!` / `update!` failed validation |
| `NoMethodError: undefined method 'x' for nil` | You used `find_by` (got `nil`) and then called a method on it |

🔎 Services often turn `RecordNotFound` into a user-friendly error, `app/services/attendance/daily_attendance_tags/imports/enqueue_service.rb:40-41`:

```ruby
    rescue ActiveRecord::RecordNotFound
      raise ::HrErrors::UserError.new('日次勤怠タグインポート情報が存在しません。', status: :not_found)
```

---

## 9. Practice (15 min)

Answer with Ruby code (no need to run it).

1. Get the comment with id 7 **belonging to** `company` (raise if not found).
2. Get all of `company`'s comments on url `'/payroll'`, newest first.
3. Get just the ids of `company`'s comments by user 5.
4. Is there any comment of `company` on url `'/home'`? (true/false)
5. Create a comment for `company` (user 5, url `'/home'`, text `'こんにちは'`) and raise if invalid.
6. Change comment `c`'s text to `'修正'` and raise if invalid.
7. Which line runs SQL? `a = company.comments.where(user_id: 5)` / `b = a.order(:id)` / `c = b.first`

*(Answers:*
1. `company.comments.find(7)`
2. `company.comments.where(url: '/payroll').order(created_at: :desc)`
3. `company.comments.where(user_id: 5).pluck(:id)`
4. `company.comments.exists?(url: '/home')` (or `.where(url: '/home').exists?`)
5. `company.comments.create!(user_id: 5, url: '/home', comment: 'こんにちは')`
6. `c.update!(comment: '修正')`
7. Only `c = b.first` (relations are lazy).)*

---

## 10. Summary

- **ActiveRecord** = Rails' ORM: model class ⇄ table, object ⇄ row, method ⇄ column.
- All models inherit `ApplicationRecord` (abstract, adds leak check + DB connections).
- **CRUD**: `new/save`, `create(!)`, `find`, `find_by`, `update(!)`, `destroy(!)`. **`destroy` = physical delete**; soft delete uses `deleted_at`.
- **Bang methods** raise on failure; freee-payroll prefers them.
- **Queries**: `where` (hash, array → IN, range, `not`, `'... ?'` placeholders), `order`, `limit`, `joins`, `count`, `exists?`, `pluck`.
- **Relations are lazy**: SQL runs only when data is needed.
- **freee rule:** query through `company.xxx` / scopes / repositories, not `Model.where(...)` (`Freee/QueryFromActiveRecordClass`).

➡️ Now take **Test11.md**.
