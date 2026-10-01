# Test 12 — ActiveRecord 2: Associations, Validations, Callbacks, Scopes

**Rules:** Don't look at Lesson12.md. Write your answers under each question, then check `Answers12.md`.
**Total:** 43 points

---

## Part A — Multiple choice (1 point each)

**1.** `Comment belongs_to :company` means:
- a) The `companies` table has a `comment_id` column
- b) The `comments` table has a `company_id` column
- c) A comment can have many companies
- d) Comments and companies share the same table

**2.** In freee-payroll, `belongs_to :company` (with no options) makes the company:
- a) Required
- b) Optional (because `belongs_to_required_by_default = false`)
- c) Forbidden
- d) Automatically created

**3.** What does `dependent: :destroy` on a `has_many` do?
- a) Prevents destroying the parent
- b) When the parent is destroyed, its children are destroyed too
- c) Soft-deletes the children
- d) Makes the association optional

**4.** Which is a *custom* validation?
- a) `validates :name, presence: true`
- b) `validate :display_period_validity`
- c) `belongs_to :company`
- d) `scope :published, -> { where(is_published: true) }`

**5.** A callback is:
- a) A phone call from a customer
- b) A method Rails automatically runs at a moment in a record's lifecycle
- c) A type of association
- d) A database index

**6.** Which callback is the safest place to enqueue a background job that reads the saved data?
- a) `before_validation`
- b) `before_save`
- c) `after_commit`
- d) `after_find`

**7.** `default_scope { where(deleted_at: DEFAULT_DELETED_AT) }` means:
- a) Every query on the model automatically excludes soft-deleted rows
- b) Rows are deleted by default
- c) The column is dropped
- d) Only deleted rows are returned

**8.** `company.employees` does NOT return:
- a) Alive employees of that company
- b) Employees of that company that have a company_reference_date_rule
- c) Employees of that company without a company_reference_date_rule
- d) Employees whose deleted_at is 1000-01-01

---

## Part B — Fill in the blanks (1 point per blank)

**9.** To require a `belongs_to` association in freee-payroll, add `______: false`.

**10.** When the association name doesn't match the class, use the option ______; when the column name doesn't match, use ______.

**11.** Inside a custom validation method, you mark a record invalid with `errors.______(:attribute, 'message')`.

**12.** A scope's body is written as a ______ (`-> { ... }`).

**13.** To remove a default-scope condition for one query, use `______(where: :column_name)`.

**14.** To get employees including those without a closing-date rule, call `company.employees.______`.

---

## Part C — Fill in the missing code (1 point per blank)

**15.**

```ruby
class Memo < ApplicationRecord
  ______ { where(deleted_at: DEFAULT_DELETED_AT) }

  ______ :company, optional: false
  ______ :memo_attachments, dependent: :destroy

  ______ :body, presence: true, length: { ______: 500 }
  ______ :check_body_not_only_spaces

  ______ :pinned, -> { where(is_pinned: true) }

  private

  def check_body_not_only_spaces
    errors.______(:body, 'は空白だけにできません') if body.present? && body.strip.empty?
  end
end
```

---

## Part D — Write the code (3 points each)

**16.** Write a model `EmployeeNickname` that:
- hides soft-deleted rows by default
- belongs to company (required) and employee (required)
- validates `nickname` is present and at most 30 characters
- validates `employee_id` is unique within `company_id` and `deleted_at`

**17.** Add to a model `Notice`:
- a scope `published` (`is_published` true)
- a scope `for_date` that takes a `date` and returns rows with `start_on <= date` and `end_on >= date` (use `?` placeholders)

**18.** Add a callback to `Memo` that runs `normalize_body` **before every save**, but only if `body` changed.

---

## Part E — Answer in a sentence (2 points each)

**19.** Why do many freee-payroll models write both `belongs_to :company` **and** `validates :company_id, presence: true`?

**20.** What's the difference between `dependent: :destroy` and `dependent: :delete_all`?

**21.** Why is it a problem to forget about Employee's second default scope (`where.not(company_reference_date_rule_id: nil)`)?

---

## Part F — Read real code (5 points)

`app/models/employee_time_clock.rb` (excerpt):

```ruby
class EmployeeTimeClock < ApplicationRecord
  default_scope { where(deleted_at: DEFAULT_DELETED_AT) }

  belongs_to :company
  belongs_to :employee
  belongs_to :employee_time_clock_daily_summary
  has_one :employee_time_clock_for_device

  validates :employee_id, presence: true, numericality: { only_integer: true }
  validates :note, length: { in: 0...255 }

  validate :check_datetime

  scope :by_summary_date,
        ->(date) do
          joins(:employee_time_clock_daily_summary).where(employee_time_clock_daily_summaries: { date: date })
        end

  scope :unscope_deleted_at, -> { unscope(where: :deleted_at) }
end
```

**22.** (1) Which columns must the `employee_time_clocks` table have for the three `belongs_to` lines?
**23.** (1) What does `validates :note, length: { in: 0...255 }` allow? (Careful with `...`.)
**24.** (1) What does `EmployeeTimeClock.by_summary_date(Date.new(2026, 9, 1))` return, in words?
**25.** (1) Why does the scope `unscope_deleted_at` exist?
**26.** (1) Where would you look to see what `check_datetime` checks?
