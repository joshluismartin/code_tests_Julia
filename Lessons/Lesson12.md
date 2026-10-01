# Lesson 12 — ActiveRecord 2: Associations, Validations, Callbacks, Scopes

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Read the "declaration" lines at the top of any freee-payroll model: `belongs_to`, `has_many`, `validates`, `before_save`, `scope`, `default_scope`, and know what each one gives you.
**You need:** Lesson 11. Open `app/models/` in your editor.

---

## 0. Macros: the lines at the top of a model

Look at any model and the first lines are like this:

```ruby
class Something < ApplicationRecord
  belongs_to :company        # association
  has_many :things           # association
  validates :name, presence: true   # validation
  before_save :do_something  # callback
  scope :active, -> { ... }  # scope
end
```

| Term | Meaning |
|---|---|
| **Macro / class macro** | A class-method call written at the class level that *adds behavior* to the class (it's just a method call without parentheses). `belongs_to :company` is really `belongs_to(:company)`. |
| **Declaration** | Code that says *what is true* ("a comment belongs to a company") rather than *step-by-step instructions*. |

---

## 1. Associations: how models connect

| Term | Meaning |
|---|---|
| **Association** | A declared relationship between two models, backed by a foreign key. |
| `belongs_to :x` | "My table has an `x_id` column pointing at an X." |
| `has_many :xs` | "Many rows in the `xs` table have **my** id in their `<me>_id` column." |
| `has_one :x` | Like `has_many`, but at most one. |
| `has_many :xs, through: :ys` | "I reach xs by going through ys" (a join via a middle table). |

### 1.1 `belongs_to` and `has_many` go in pairs

```
companies                      comments
┌────┬──────┐                 ┌────┬────────────┬─────┐
│ id │ name │ ◀─────────────  │ id │ company_id │ ... │
└────┴──────┘   company_id    └────┴────────────┴─────┘
Company has_many :comments     Comment belongs_to :company
```

🔎 `app/models/comment.rb:2`: `belongs_to :company`
🔎 `app/models/company.rb:353`: `has_many :comments`

What each gives you (illustrative):

```ruby
comment.company            # the Company object (SELECT ... FROM companies WHERE id = comment.company_id)
company.comments           # a relation of Comments (WHERE company_id = company.id)
company.comments.create!(...)   # creates a comment with company_id already filled in
company.comments.count
```

### 1.2 A real model with several associations

🔎 `app/models/company_health_insurance_setting.rb:1-11`:

```ruby
# typed: strict

class CompanyHealthInsuranceSetting < ApplicationRecord
  belongs_to :company
  has_many :company_health_insurance_setting_revisions
  has_many :employee_health_insurance_rules

  validates :name, presence: true, length: { maximum: 240 }
  validates :is_disabled, inclusion: [true, false]
  validates :is_default, inclusion: [true, false]
end
```

Read it in words: "A company health-insurance setting belongs to one company; it has many revisions and many employee health-insurance rules. Its name is required and at most 240 characters; `is_disabled` and `is_default` must be true or false."

### 1.3 Association options

🔎 `app/models/employee_time_clock.rb:4-8`:

```ruby
  belongs_to :company
  belongs_to :employee
  belongs_to :employee_time_clock_daily_summary
  has_one :employee_time_clock_for_device
  has_one :time_clock_app_action_log
```

🔎 `app/models/attendance/check_status.rb:4-8`:

```ruby
  belongs_to :company
  has_many :attendance_check_results,
           class_name: 'Attendance::CheckResult',
           foreign_key: :attendance_check_status_id,
           dependent: :delete_all
```

| Option | Meaning |
|---|---|
| `class_name: 'Attendance::CheckResult'` | The association name doesn't match the class name, so say which class. |
| `foreign_key: :attendance_check_status_id` | Which column holds the link (default would be `check_status_id`). |
| `dependent: :destroy` | When I'm destroyed, destroy my children too (runs their callbacks). |
| `dependent: :delete_all` | When I'm destroyed, delete children with one SQL `DELETE` (no callbacks, faster). |
| `through: :ys` | Reach through another association. |
| `-> { ... }` (a lambda before options) | Add extra conditions to the association. |

🔎 `has_many ... through:`, `app/models/company_reference_date_rule.rb:23`:

```ruby
  has_many :doc_output_logs, through: :company_payroll_book_logs
```

🔎 An association with a lambda, `app/models/user_employee.rb:14`:

```ruby
  belongs_to :employee, -> { unscope_company_reference_date_rule_id }
```

(This removes one of Employee's default conditions when loading `user_employee.employee`; we'll understand why in §5.)

### 1.4 ⚠️ freee-payroll rule: `belongs_to` is optional by default

In normal Rails 5+, `belongs_to :company` **requires** a company: saving without one fails. freee-payroll turned that off.

🔎 `config/application.rb:174`:

```ruby
    Rails.application.config.active_record.belongs_to_required_by_default = false
```

So in this repo:

| You write | Meaning here |
|---|---|
| `belongs_to :company` | Company is **optional** (a nil company is allowed) |
| `belongs_to :company, optional: false` | Company is **required** |

🔎 `app/models/employee_holiday_pool.rb:4-5`:

```ruby
  belongs_to :company, optional: false
  belongs_to :employee, optional: false
```

That's why many models *also* write `validates :company_id, presence: true` (e.g. `comment.rb:4`): to require it the explicit way.

---

## 2. Validations: rules before saving

| Term | Meaning |
|---|---|
| **Validation** | A rule checked before saving. If any rule fails, the record is **invalid** and isn't saved. |
| `valid?` | Runs validations; returns true/false. |
| `errors` | After validation, holds the failure messages (`record.errors.full_messages`). |

### 2.1 Built-in validators

| Validator | Example | Rule |
|---|---|---|
| `presence: true` | `validates :url, presence: true` | not nil / not blank |
| `length:` | `length: { maximum: 255 }`, `{ in: 0...255 }`, `{ is: 4 }` | text length |
| `numericality:` | `numericality: { only_integer: true, greater_than: -1 }` | must be a number |
| `inclusion:` | `inclusion: [true, false]`, `inclusion: %w[floor round ceil]` | value must be in the list |
| `format:` | `format: { with: /\A\d{4}\z/ }` | must match a regular expression |
| `uniqueness:` | `uniqueness: { scope: %i[company_id deleted_at] }` | no other row with the same value (within scope) |
| `allow_blank: true` | | skip the check when blank |
| `if:` / `unless:` | `if: -> { link_url.present? }` | only check under a condition |

🔎 `app/models/employee_time_clock.rb:26-32`:

```ruby
  validates :company_id, presence: true, numericality: { only_integer: true }
  validates :employee_id, presence: true, numericality: { only_integer: true }
  validates :employee_time_clock_daily_summary, presence: true
  validates :type, presence: true
  validates :datetime, presence: true
  validates :original_datetime, presence: true
  validates :note, length: { in: 0...255 }
```

🔎 A uniqueness rule that includes `deleted_at`, `app/models/user_employee.rb:21`:

```ruby
  validates :employee_id, uniqueness: { scope: %i[company_id deleted_at] }
```

"Within the same company and the same `deleted_at`, an `employee_id` may appear only once." Since alive rows all have `deleted_at = 1000-01-01`, only **one alive** row per employee is allowed, while many deleted ones can exist.

### 2.2 Custom validations: `validate` (singular) + `errors.add`

🔎 `app/models/banner.rb:22` and `:34-40`:

```ruby
  validate :display_period_validity
  ...
  def display_period_validity
    return if display_start_at.blank? || display_end_at.blank?

    if display_start_at && display_end_at && display_start_at > display_end_at
      errors.add(:display_end_at, 'は開始日時より後に設定してください')
    end
  end
```

| Code | Meaning |
|---|---|
| `validate :method_name` | "Also run this method during validation." (Note: `validate`, not `validates`.) |
| `errors.add(:display_end_at, '...')` | Mark the record invalid with a message on that attribute. Messages are Japanese and join with the attribute's name: "表示終了日時は開始日時より後に..." |
| `errors.add(:base, '...')` | An error not tied to a single attribute. |

🔎 `app/models/employee_time_clock.rb:34` + `:65-71` does the same with `errors.add(:base, '出勤日時と勤務起算日が異なります。')`.

### 2.3 Using validations

```ruby
# (illustrative example)
c = company.comments.new(user_id: 1, url: '', comment: 'x')
c.valid?                 # => false
c.errors.full_messages   # => ["Urlを入力してください"] (approximate)
c.save                   # => false
c.save!                  # raises ActiveRecord::RecordInvalid
```

---

## 3. Callbacks: code that runs automatically

| Term | Meaning |
|---|---|
| **Callback** | A method Rails calls automatically at a certain moment in a record's life (before saving, after creating...). |
| **Lifecycle** | The sequence: initialize → validate → save (create or update) → commit; or destroy. |

| Callback | Runs... |
|---|---|
| `before_validation` | before validations |
| `before_save` / `after_save` | around every save (create **and** update) |
| `before_create` / `after_create` | only when inserting a new row |
| `before_update` / `after_update` | only when updating an existing row |
| `before_destroy` / `after_destroy` | around `destroy` |
| `after_commit` | after the DB transaction is committed (safe for side effects like notifications) |
| `after_find` | after loading a record from the DB |

🔎 `app/models/slack_integration.rb:9-13`:

```ruby
  after_create :create_token_parameter, :create_slack_team_parameter
  after_find :decrypt_token

  before_save :clear_token_if_inactive
  after_update :encrypt_token, :update_slack_team_parameter
```

"After creating: create the token parameter rows. After loading: decrypt the token. Before any save: clear the token if inactive. After updating: encrypt the token and update the team info."

🔎 Conditional callbacks, `app/models/company_health_insurance_pay_rule.rb:25-26`:

```ruby
  before_validation :set_default_kumiai_type, if: :invalid_kumiai?
  before_validation :nullify_kumiai_type, unless: :kumiai_category?
```

🔎 Only when a column changed, `app/models/employee.rb:180`:

```ruby
  before_update :on_update_company_reference_date_rule_id, if: :company_reference_date_rule_id_changed?
```

(`x_changed?` is an automatic "dirty tracking" method: true if `x` was modified since loading.)

🔎 `after_commit`, `app/models/employee_payroll_statement.rb:196`:

```ruby
  after_commit :enqueue_async_calc
```

"Once the employee's payroll statement is really committed to the DB, enqueue a background payroll calculation." Using `after_commit` (not `after_save`) matters here: a background job runs in a *different* process, so it must only start when the data is committed and visible to it.

> **Reading tip:** callbacks are "invisible code." When a save does something surprising, look at the model's `before_*` / `after_*` lines. Also remember `update_all` / `delete_all` **skip** callbacks and validations.

---

## 4. Scopes: named, reusable queries

| Term | Meaning |
|---|---|
| **Scope** | A named query defined on the model: `scope :name, -> { where(...) }`. Calling it returns a relation, so it chains. |
| **Lambda `-> { }`** | A small anonymous function (Lesson 5). Scopes use it so the query is built fresh each time. |

🔎 `app/models/banner.rb:24-30`:

```ruby
  scope :published, -> { where(is_published: true) }
  scope :current,
        -> do
          now = Time.current
          where('display_start_at <= ? AND display_end_at >= ?', now, now)
        end
  scope :for_all_roles, -> { where(is_admin_only: false) }
```

Using them (illustrative): `Banner.published.current.for_all_roles` → published banners showing right now for everyone.

🔎 Scopes with arguments, `app/models/employee_holiday_pool.rb:11-18`:

```ruby
  scope :available, -> { where('employee_holiday_pools.left > 0') }
  scope :available_at,
        ->(date) do
          where('employee_holiday_pools.left > 0').where('valid_from <= ?', date.to_date).where(
            'valid_to >= ?',
            date.to_date,
          )
        end
```

`->(date) do ... end` takes an argument: `employee.employee_holiday_pools.available_at(Date.current)` = paid-leave pools with days left that are valid today.

🔎 A scope with a `joins`, `app/models/employee_time_clock.rb:36-39`:

```ruby
  scope :by_summary_date,
        ->(date) do
          joins(:employee_time_clock_daily_summary).where(employee_time_clock_daily_summaries: { date: date })
        end
```

`where(other_table: { column: value })` filters on a **joined** table's column.

Scopes chain with associations too: `company.employees.accessible.exists?(id: ...)` uses the `accessible` scope on the `employees` association.

---

## 5. `default_scope` and `unscope`

| Term | Meaning |
|---|---|
| **`default_scope`** | A condition added **automatically to every query** on the model. |
| **`unscope(where: :column)`** | Remove a condition (e.g. from the default scope) for this one query. |

🔎 `app/models/employee.rb:29-30`:

```ruby
  default_scope { where(deleted_at: DEFAULT_DELETED_AT) }
  default_scope { where.not(company_reference_date_rule_id: nil) }
```

So `company.employees` actually means: *alive* employees (`deleted_at = 1000-01-01`) **that have a closing-date/pay-date rule set** (`company_reference_date_rule_id IS NOT NULL`, CRDR = 締め日・支払日 rule).

And to see the rows the default scopes hide, the model provides scopes:

🔎 `app/models/employee.rb:325`, `:543`, `:546`:

```ruby
  scope :include_deleted, -> { unscope(where: :deleted_at) }
  ...
  scope :unscope_company_reference_date_rule_id, -> { unscope(where: :company_reference_date_rule_id) }
  ...
  scope :without_payroll, -> { unscope_company_reference_date_rule_id.where(company_reference_date_rule_id: nil) }
```

| You want | Write |
|---|---|
| alive employees with CRDR (normal) | `company.employees` |
| also employees without CRDR (not in payroll) | `company.employees.unscope_company_reference_date_rule_id` |
| only employees without CRDR | `company.employees.without_payroll` |
| also deleted employees | `company.employees.include_deleted` |

> ⚠️ `app/README.md` warns about this: forgetting that Employee's default scope hides employees without a CRDR is a classic freee-payroll bug. That's why `UserEmployee`'s association (`user_employee.rb:14`) and `MemberRepository#employee_scope` both call `unscope_company_reference_date_rule_id`.

---

## 6. Putting it together: read a whole model

🔎 `app/models/user_employee.rb:10-37`:

```ruby
class UserEmployee < ApplicationRecord
  default_scope { where(deleted_at: DEFAULT_DELETED_AT) }

  belongs_to :company
  belongs_to :employee, -> { unscope_company_reference_date_rule_id }

  before_destroy :clear_bumon_manager_relations

  validates :company_id, numericality: { only_integer: true }
  validates :employee_id, numericality: { only_integer: true }
  validates :auth_user_id, numericality: { only_integer: true }
  validates :employee_id, uniqueness: { scope: %i[company_id deleted_at] }

  enum :status, { invited: 'invited', registered: 'registered', signed_up: 'signed_up' }

  private

  # 部門管理者としての設定を解除
  # 従業員が再利用され他のアカウントに紐付いたときに部門管理者としての設定が再利用されないように解除する。
  def clear_bumon_manager_relations
    return if employee&.member_id.blank?

    # 呼び出し元セッションのアクセス制限とは無関係に、解除対象の関係は必ず消す必要があるため leakcheck を無効化する
    Authz.without_employee_leakcheck do
      company.bumon_managers_managing_relations.where(member_id: employee.member_id).destroy_all
    end
  end
end
```

In words:
- It links a **user account** to an **employee** (see the file's top comment).
- Default: only alive rows.
- Belongs to a company and an employee (including employees without CRDR).
- **Before destroying**, it removes department-manager relations.
- Ids must be integers; only one alive link per employee per company.
- `status` is an **enum** (Lesson 13): invited / registered / signed_up.
- `Authz.without_employee_leakcheck` is explained in Lesson 13.

---

## 7. Practice (15 min)

**Practice 1.** Write a model `Memo` (table `memos`: `id, company_id, employee_id, body, deleted_at, ...`) that:
- belongs to company (required) and employee
- validates `body` is present and at most 500 characters
- hides deleted rows by default
- has a scope `recent` ordering by `created_at` descending

**Practice 2.** In words, what does this do? (`app/models/company_reference_date_rule.rb:26`)

```ruby
  validates :closing_day, inclusion: 1..31, uniqueness: { scope: %i[company_id pay_day month_of_pay_day deleted_at] }
```

**Practice 3.** How do you get a company's employees **including** those without a closing-date rule?

*(Answers:*

```ruby
class Memo < ApplicationRecord
  default_scope { where(deleted_at: DEFAULT_DELETED_AT) }

  belongs_to :company, optional: false
  belongs_to :employee

  validates :body, presence: true, length: { maximum: 500 }

  scope :recent, -> { order(created_at: :desc) }
end
```

*Practice 2: closing_day must be 1–31, and within the same company + pay_day + month_of_pay_day + deleted_at there can be only one rule with that closing day (only one alive duplicate). Practice 3: `company.employees.unscope_company_reference_date_rule_id`.)*

---

## 8. Summary

- **Associations**: `belongs_to` (I hold the FK), `has_many` / `has_one` (they hold my id), `through:`, options `class_name:`, `foreign_key:`, `dependent:`.
- ⚠️ freee: **`belongs_to` is optional by default**; use `optional: false` (or `validates :x_id, presence: true`) to require it.
- **Validations**: `validates :attr, presence/length/numericality/inclusion/format/uniqueness`; custom `validate :method` + `errors.add`.
- **Callbacks**: `before_/after_` `validation/save/create/update/destroy/commit/find`. Hidden side effects: check them when saves surprise you.
- **Scopes**: named lambdas returning relations; chainable.
- **`default_scope`** applies to every query; **Employee** has two (not deleted + CRDR present). Use `unscope_company_reference_date_rule_id` / `include_deleted` when needed.

➡️ Now take **Test12.md**.
