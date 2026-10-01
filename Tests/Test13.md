# Test 13 — ActiveRecord 3: freee-payroll-Specific Model Rules

**Rules:** Don't look at Lesson13.md. Write your answers under each question, then check `Answers13.md`.
**Total:** 47 points

(`Memo` / `memos` below is an illustrative model with a `deleted_at` column and `belongs_to :employee`.)

---

## Part A — Multiple choice (1 point each)

**1.** Which of these is a **physical** delete?
- a) `memo.destroy_logically!`
- b) `memo.update!(deleted_at: Time.current)`
- c) `memo.destroy`
- d) `memos.destroy_all_logically!`

**2.** Where are `destroy_logically!` and `destroy_all_logically!` defined?
- a) In every model file
- b) `config/initializers/logical_delete.rb` (added to `ActiveRecord::Base`)
- c) `db/schema.rb`
- d) A gem called `paranoia`

**3.** What is a concern?
- a) A bug report
- b) A module using `ActiveSupport::Concern` that packages shared model code
- c) A kind of migration
- d) A database trigger

**4.** Inside a concern, `included do ... end` is for:
- a) Code that runs as if written inside the including class (e.g. `validates`, `scope`)
- b) Tests
- c) Code that runs only once at boot
- d) Private methods only

**5.** `enum :round_type, { floor: 'floor', round: 'round', ceil: 'ceil' }, prefix: true` generates:
- a) `floor?`
- b) `round_type_floor?`
- c) `prefix_floor?`
- d) nothing

**6.** In aasm, what is an "event"?
- a) A named action that moves the record from one state to another
- b) A database column
- c) A JavaScript click
- d) A log message

**7.** What does LeakCheck protect against?
- a) Memory leaks
- b) Accessing/saving records whose `employee_id` the current user is not allowed to access
- c) SQL injection
- d) Leaking passwords in logs

**8.** The N+1 problem is:
- a) Having N+1 tables
- b) Running 1 query for a list, then 1 more query per item
- c) An off-by-one bug in loops
- d) A MySQL error code

**9.** Which is NOT a good place for `DatabaseSelector.connected_to_reader`?
- a) A heavy read-only report query
- b) A repository read method
- c) Reading a record you just wrote, inside a transaction
- d) A list API that only reads

**10.** `find_each(batch_size: 1000)`:
- a) Loads all records at once
- b) Iterates over all records, loading 1,000 at a time
- c) Returns only the first 1,000
- d) Deletes records in batches

---

## Part B — Fill in the blanks (1 point per blank)

**11.** To eager-load an association and avoid N+1, you can use ______, ______, or ______.

**12.** For 1,000+ records, freee-payroll says to avoid `to_a` and use ______ (for plain values) or ______ (for iterating records).

**13.** To intentionally ignore LeakCheck for a block, wrap it in `Authz.______ do ... end`, and you must write a ______ explaining why.

**14.** List queries of employees should go through the ______ scope.

**15.** All writes inside `ActiveRecord::Base.______ do ... end` are rolled back if an exception is raised.

---

## Part C — Fill in the missing code (1 point per blank)

**16.**

```ruby
module Memos::BodyValidations
  extend ActiveSupport::______

  ______ do
    validates :body, presence: true, length: { maximum: 500 }
  end

  ______ do
    def max_body_length
      500
    end
  end
end

class Memo < ApplicationRecord
  ______ Memos::BodyValidations
end
```

**17.**

```ruby
class ImportRequest < ApplicationRecord
  include AASM

  aasm(:status) do
    state :pending, ______: true
    state :running
    state :done

    ______ :start do
      transitions from: :pending, to: :______
    end

    event :finish do
      ______ from: :running, to: :done
    end
  end
end
```

---

## Part D — Write the code (3 points each)

**18.** Soft-delete all of `employee`'s memos, then create a new memo for the employee (company from `employee.company_id`, body `'再作成'`), **all or nothing**.

**19.** Rewrite both snippets for memory safety (the company may have 10,000 employees). First, call `notify` on every employee without loading them all at once:
```ruby
company.employees.to_a.each { |e| notify(e) }
```
And separately, get only the ids of alive memos of `company`:
```ruby
company.memos.where(deleted_at: nil).to_a.map(&:id)
```

**20.** Fix the N+1 and use the replica for this read-only report:
```ruby
rows = company.employees.map { |e| [e.id, e.company_reference_date_rule&.name] }
```

---

## Part E — Answer in a sentence (2 points each)

**21.** What's the difference between `relation.destroy_all_logically!` and calling `destroy_logically!` on each record?

**22.** Why must you not use the read replica to read data you just wrote?

**23.** Why does `company_daily_prorate_setting.rb` use `prefix: true` on its enum?

---

## Part F — Read real code (5 points)

`app/models/user_employee.rb`:

```ruby
class UserEmployee < ApplicationRecord
  default_scope { where(deleted_at: DEFAULT_DELETED_AT) }

  belongs_to :company
  belongs_to :employee, -> { unscope_company_reference_date_rule_id }

  before_destroy :clear_bumon_manager_relations

  validates :employee_id, uniqueness: { scope: %i[company_id deleted_at] }

  enum :status, { invited: 'invited', registered: 'registered', signed_up: 'signed_up' }

  private

  def clear_bumon_manager_relations
    return if employee&.member_id.blank?

    # 呼び出し元セッションのアクセス制限とは無関係に、解除対象の関係は必ず消す必要があるため leakcheck を無効化する
    Authz.without_employee_leakcheck do
      company.bumon_managers_managing_relations.where(member_id: employee.member_id).destroy_all
    end
  end
end
```

**24.** (1) When does `clear_bumon_manager_relations` run?
**25.** (1) Why is LeakCheck disabled inside it? (Translate/explain the comment.)
**26.** (1) What does `employee&.member_id.blank?` protect against?
**27.** (1) Name two methods the `enum :status` line generates.
**28.** (1) If a `UserEmployee` is soft-deleted with `destroy_logically!`, does `clear_bumon_manager_relations` run? Why or why not?
