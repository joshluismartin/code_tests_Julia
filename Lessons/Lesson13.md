# Lesson 13 — ActiveRecord 3: freee-payroll-Specific Model Rules

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Know the model-level rules that are *special to freee-payroll* (soft delete, concerns, enums/state machines, LeakCheck, memory and N+1 rules, transactions, read replica) so you can read and write model code safely.
**You need:** Lessons 10–12. Open `app/README.md` (it's the source of most rules here).

---

## 1. Soft delete in models

Recap from Lesson 10: `deleted_at = DEFAULT_DELETED_AT` (`Time.utc(1000, 1, 1)`, `config/constants.rb:51`) = **alive**.

### 1.1 Hiding deleted rows: `default_scope`

🔎 88 models in `app/models/` contain exactly this line, e.g. `app/models/employee_time_clock.rb:2`:

```ruby
  default_scope { where(deleted_at: DEFAULT_DELETED_AT) }
```

→ `company.employee_time_clocks` never returns deleted rows unless you `unscope(where: :deleted_at)`.

### 1.2 Deleting softly

🔎 `config/initializers/logical_delete.rb:1-13` adds two helpers to **every** model:

```ruby
module ActiveRecord
  class Base
    def destroy_logically!
      self.deleted_at = Time.zone.now
      self.save!
    end

    def self.destroy_all_logically!
      time_now = Time.zone.now
      update_all(deleted_at: time_now, updated_at: time_now)
    end
  end
end
```

| Term | Meaning |
|---|---|
| **Initializer** | A file in `config/initializers/` that runs once when the app boots. |
| **Monkey patch** | Adding/changing methods of an existing class (here, Rails' own `ActiveRecord::Base`) from outside it. Powerful but surprising, so this repo keeps them in initializers. |

| Method | What it does |
|---|---|
| `record.destroy_logically!` | Set `deleted_at` to now and `save!` (runs validations/callbacks). |
| `relation.destroy_all_logically!` | One SQL `UPDATE ... SET deleted_at = now` for all rows (**skips** validations/callbacks). |
| `record.update(deleted_at: Time.current)` | The plain way (`app/README.md` describes soft delete like this). |
| `destroy` / `delete_all` | ❌ **physical** delete, not a soft delete. |

🔎 A model can customize it, `app/models/employee.rb:729-732`:

```ruby
  def destroy_logically!
    self.member_id = nil
    super
  end
```

"When soft-deleting an employee, also clear `member_id`, then do the normal soft delete (`super`)."

🔎 Used on associations, `app/models/employee.rb:686`: `sanzen_sango_ikuji_kyugyos.destroy_all_logically!`

### 1.3 The three soft-delete mistakes

| ❌ Wrong | ✅ Right | Why |
|---|---|---|
| `where(deleted_at: nil)` | `where(deleted_at: DEFAULT_DELETED_AT)` (or rely on `default_scope`) | nil never occurs |
| `record.destroy` to "delete" a soft-deletable row | `record.destroy_logically!` / `update(deleted_at: Time.current)` | destroy removes history |
| New table without `deleted_at` default | `t.datetime :deleted_at, null: false, default: '1000-01-01 00:00:00'` | keeps the convention |

---

## 2. Concerns: sharing model code

| Term | Meaning |
|---|---|
| **Concern** | A Ruby **module** (Lesson 7) that uses `ActiveSupport::Concern` to package model code (validations, scopes, methods) so several models can `include` it. Lives in `app/models/concerns/`. |
| `extend ActiveSupport::Concern` | Turns a plain module into a concern (enables `included do` and `class_methods do`). |
| `included do ... end` | Code that runs **inside the including class** as if written there (good for `validates`, `scope`, `belongs_to`...). |
| `class_methods do ... end` | Methods that become **class methods** of the including class. |
| Instance methods | Plain `def`s in the module become instance methods. |

🔎 `app/models/concerns/company_rules/bank_account_validations.rb:1-22`:

```ruby
module CompanyRules::BankAccountValidations
  extend ActiveSupport::Concern

  included do
    validates :company_code, numericality: { only_integer: true }, allow_blank: true, length: { is: 10 }

    # 振込依頼人名はカタカナと英字のみ許容する
    validates :company_name,
              length: {
                maximum: 40,
              },
              allow_blank: true,
              format: {
                with: /\A[A-Za-zＡ-Ｚａ-ｚ\p{katakana}\p{blank}ー－\-ｰ()（）゛ﾞ゜ﾟ]+\z/,
              }
    validates :bank_code, length: { is: 4 }, allow_blank: true, numericality: { only_integer: true }
    validates :branch_code, length: { is: 3 }, allow_blank: true, numericality: { only_integer: true }
    validates :account_number, length: { is: 7 }, allow_blank: true, numericality: { only_integer: true }
    validates :type_code, allow_blank: true, inclusion: %w[general payroll bonus]
    validates :new_code, inclusion: [0, 1, 2], numericality: { only_integer: true }
  end
end
```

Used by 🔎 `app/models/company_bank_account_pay_rule.rb:4`: `include CompanyRules::BankAccountValidations`. Now that model gets all those bank-account rules (bank code = 4 digits, branch = 3, account = 7, the transfer name is katakana/English only).

🔎 `class_methods do`, `app/models/concerns/company_reference_date_rule/reimbursed_expense_cps_finder.rb:55-79` (used by `company_reference_date_rule.rb:15`):

```ruby
  class_methods do
    ...
    def bulk_find_or_create_cps_with_upcoming_pay_date(rule_ids, date)
      return {} if rule_ids.blank?
      ...
      rules = where(id: rule_ids).index_by(&:id)
      ...
    end
  end
```

→ callable as `CompanyReferenceDateRule.bulk_find_or_create_cps_with_upcoming_pay_date(ids, date)`.

---

## 3. Enums: named values for a column

### 3.1 Rails `enum`

| Term | Meaning |
|---|---|
| **enum** | Declares that a column can only hold a fixed set of named values, and generates helper methods. |

🔎 `app/models/user_employee.rb:23`:

```ruby
  enum :status, { invited: 'invited', registered: 'registered', signed_up: 'signed_up' }
```

Generated helpers (illustrative):

```ruby
ue.invited?          # status == 'invited'
ue.registered!       # update status to 'registered' (saves)
UserEmployee.invited # scope: where(status: 'invited')
UserEmployee.statuses # => { "invited" => "invited", ... }
```

🔎 With `prefix: true`, `app/models/company_daily_prorate_setting.rb:5-12`:

```ruby
  enum :round_type,
       {
         # 日割り計算の端数処理方法
         floor: 'floor', # 切り捨て
         round: 'round', # 四捨五入
         ceil: 'ceil', # 切り上げ
       },
       prefix: true
```

`prefix: true` names the helpers `round_type_floor?`, `round_type_round!`... (a bare `round` would clash with Ruby's own `round` method!).

### 3.2 `enumerize` (a gem doing something similar)

🔎 `app/models/e_file_yearend_remark.rb:7` and `:11`:

```ruby
  extend Enumerize
  ...
  enumerize :type, in: %i[etax eltax], predicates: true, scope: true
```

| Option | Gives you |
|---|---|
| `in: %i[etax eltax]` | allowed values |
| `predicates: true` | `remark.etax?`, `remark.eltax?` |
| `scope: true` | `EFileYearendRemark.with_type(:etax)` |

Both styles exist in the repo. When you see `enum` → Rails; `enumerize` → the gem.

### 3.3 State machines with `aasm`

| Term | Meaning |
|---|---|
| **State machine** | A model whose `status` can only move between states along allowed **transitions**, triggered by **events**. |
| **State** | A possible value (e.g. `standby`). |
| **Event** | A named action that moves the state (`check`). |
| **Transition** | "From state A to state B." |

🔎 `app/models/attendance/check_status.rb:1-2`, `:13-29` (attendance check batch):

```ruby
class Attendance::CheckStatus < ApplicationRecord
  include AASM
  ...
  aasm(:status) do
    state :standby, initial: true
    state :checking
    state :check_finished
    ...
    event :check do
      transitions from: :standby, to: :checking
    end

    event :checked do
      transitions from: :checking, to: :check_finished
    end
```

```
standby ──check──▶ checking ──checked──▶ check_finished ──report──▶ reporting ──reported──▶ report_finished
```

Generated helpers (illustrative): `cs.standby?`, `cs.may_check?`, `cs.check!` (changes state and saves). Calling `check!` from the wrong state raises an error. The state machine prevents impossible jumps.

---

## 4. LeakCheck: never show employees you're not allowed to see

| Term | Meaning |
|---|---|
| **Authorization / 権限** | What the logged-in user is *allowed* to do/see. |
| **Accessible employees** | The employees the current user may access (e.g. a department manager, 部門管理者, only sees their department). |
| **LeakCheck** | freee-payroll's safety net: if code loads or saves a record whose `employee_id` is **outside** the accessible set, it raises an error instead of leaking data. |

🔎 Set up once for every model, `app/models/application_record.rb:4` and `:7-16`:

```ruby
  include OmegaRails::Model::LeakCheckable
  ...
  omega_leak_check(:raise, :loose)
  omega_leak_check_policy(
    :employee_id,
    {
      action: :raise,
      mode: :loose,
      context_value_type: :set,
      context_value: lambda { Omega::Context.current.custom[:accessible_employee_ids] },
    },
  )
```

"For any record with an `employee_id` column: if that id isn't in the current request's `accessible_employee_ids`, **raise**."

### 4.1 Do it right: the `accessible` scope

🔎 `app/models/application_record.rb:53-60` defines a helper; `app/models/employee.rb:32` uses it: `has_accessible_scope employee_id_column: :id # defineds scope :accessible`

```ruby
    def has_accessible_scope(employee_id_column:)
      scope :accessible,
            # @param [AccessScope] access_scope アクセス可能な範囲
            #   TODO: @C-FO/hr-kengen デフォルト値を削除し AccessScope の明示を強制する
            ->(access_scope = Authz.access_scope) do
              access_scope.filter_relation(relation: all, employee_id_column: employee_id_column)
            end
    end
```

🔎 In use, `app/models/company_payroll_statement.rb:290`:

```ruby
        company.employees.accessible.working_at_cps(self)
```

`app/README.md` rule: **list queries go through the `accessible` scope**, so you only fetch what the user may see and LeakCheck never fires.

### 4.2 On purpose: `Authz.without_employee_leakcheck`

Sometimes a process *must* touch all employees regardless of who's logged in (e.g. clean-up when deleting). Then you disable the check **for one block** and **write why in a comment**.

🔎 `lib/authz.rb:68-79` (comment translated: "Temporarily disable the employee leakcheck while running the block; outside the block it's enabled again"):

```ruby
    def without_employee_leakcheck(&block)
      Omega::Context.run_as(custom: omega_context_custom_leakcheck_off_params, &block)
    end
```

🔎 `app/models/user_employee.rb:32-35`:

```ruby
    # 呼び出し元セッションのアクセス制限とは無関係に、解除対象の関係は必ず消す必要があるため leakcheck を無効化する
    Authz.without_employee_leakcheck do
      company.bumon_managers_managing_relations.where(member_id: employee.member_id).destroy_all
    end
```

(Comment: "These relations must always be removed regardless of the caller session's access restrictions, so we disable leakcheck.") ← That comment is **required** by the rule.

There's also `Authz.disable_access_control { }` (`lib/authz.rb:64`), which is stronger: it turns off both the leak check and the accessible-employee restriction. You'll see it in controllers like `company_daily_prorate_settings_controller.rb` (Lesson 15).

---

## 5. Performance: N+1, preload/includes, memory

### 5.1 The N+1 problem

| Term | Meaning |
|---|---|
| **N+1 query problem** | You load N records with 1 query, then loop and trigger **1 more query per record** → N+1 queries. Very slow for 1,000 employees. |
| **Eager loading** | Loading associated records in advance, in a few big queries. |
| `preload(:assoc)` | Eager load with a **separate** query per association. |
| `includes(:assoc)` | Eager load; Rails chooses separate queries or a JOIN. |
| `eager_load(:assoc)` | Eager load with a **JOIN** (one big query). |

```ruby
# ❌ N+1 (illustrative example)
company.comments.each { |c| puts c.company.name }        # 1 + N queries

# ✅ eager loading
company.comments.preload(:company).each { |c| puts c.company.name }  # 2 queries
```

🔎 `app/modules/payroll/infrastructure/query_services/fetch_employee_payroll_statement_templates_query_service.rb:15-26`:

```ruby
            company
              .employees
              .unscope_company_reference_date_rule_id
              .where(id: employee_ids)
              .includes(employee_payroll_statement_template: :payroll_statement_template)
              .map do |employee|
                template = employee.employee_payroll_statement_template&.payroll_statement_template
                {
                  employee_id: employee.id,
                  payroll_statement_template: template ? { id: template.id, name: template.name } : nil,
                }
              end
```

`includes(a: :b)` = load `a` **and** `a`'s `b` in advance (nested). Without it, the `map` would query twice per employee.

🔎 Also `app/models/employee_holiday_pool.rb:66`: `.preload(employee_work_record: :employee_work_record_summary)`.

In development, the **bullet** gem (in `Gemfile`) warns you about N+1s.

### 5.2 Memory rule: don't load everything

🔎 `app/README.md`: *"1000件超のレコードを扱う処理は全件ロード（to_a・preload 後の Ruby select）を避け、pluck / find_each(batch_size: 1000) を使う"* = "For processes handling 1,000+ records, avoid loading everything (`to_a`, Ruby `select` after `preload`); use `pluck` / `find_each(batch_size: 1000)`." Details: `docs/knowledge/general/coding-standards/memory-optimization.md`.

| ❌ Heavy | ✅ Light | Why |
|---|---|---|
| `company.employees.to_a.map(&:id)` | `company.employees.pluck(:id)` | no Ruby objects built |
| `company.employees.each { ... }` (5,000 employees) | `company.employees.find_each(batch_size: 1000) { ... }` | loads 1,000 at a time |
| `preload(...).select { Ruby filter }` | filter in SQL (`where`, `joins` + `pluck`) | only load what you need |
| sorting in Ruby **before** paging | fetch ids + sort key first, page, then load 50 | from the memory doc |
| `.reduce({}, :merge)` | `each_with_object({})` or `.to_h` | avoids O(n²) hashes |

| Term | Meaning |
|---|---|
| **Batch** | A chunk of records processed together. |
| `find_each(batch_size: n)` | Iterates all records, loading `n` at a time (by id order). |
| `in_batches(of: n)` | Gives you relations of `n` rows, e.g. for `update_all`. |
| **OOM** (out of memory) | The process uses too much memory and crashes. |

🔎 Real `find_each`, `app/repositories/payroll/shinzoku_repository.rb:242`: `eyas.find_each(batch_size: 100) do |eya|`. (Note the `# FIXME: N+1問題が発生するので要調整` comment above it: "FIXME: N+1 happens, needs adjustment." You will meet real FIXMEs like this!)

It's OK to `to_a` small, bounded data (master data like departments, or after `limit(100)`), per the memory doc.

---

## 6. Transactions

🔎 `app/models/paid_holiday.rb:20-32`:

```ruby
  def self.import_holidays(company, import_no)
    imports = []
    ActiveRecord::Base.transaction do
      company
        .paid_holiday_imports
        .where(import_no: import_no)
        .each do |tmp|
          imports << self.new(tmp.paid_holiday_attributes)
          tmp.touch(:deleted_at)
        end
      self.import imports
    end
  end
```

| Code | Meaning |
|---|---|
| `ActiveRecord::Base.transaction do ... end` | Everything inside commits together. If an exception is raised inside, **all writes are rolled back**. |
| `tmp.touch(:deleted_at)` | Set `deleted_at` (and `updated_at`) to now: a quick soft delete. |
| `self.import imports` | Bulk INSERT (from the `activerecord-import` gem). |

Why a transaction here? Importing = "insert real rows AND mark temp rows deleted." Doing only half would duplicate or lose data.

Rule of thumb in this repo: **writes to several tables → wrap them in a transaction**, usually in the service (use case) layer (e.g. `update_company_daily_prorate_setting_service.rb:22`). This is also why bang methods matter: an exception triggers the rollback.

---

## 7. Read replica: `DatabaseSelector.connected_to_reader`

🔎 `lib/database_selector.rb:28` + `:47-57`:

```ruby
    def connected_to_reader(enable_write_forwarding: false, &)
      ...
        begin
          ActiveRecord::Base.connected_to(shard: :default, role: :reading, &)
        rescue ActiveRecord::ReadOnlyError
          ::ReachableChecker.execute(message: 'リードレプリカに書き込もうとしています。')

          # 書き込みクエリがあった場合は writing に向ける
          yield
```

"Run the block on the **read replica**. If the block tries to write (`ReadOnlyError`), report it ('trying to write to the read replica') and **re-run** the block on the primary."

🔎 Usage, `app/presenters/generate_income_tax_fixed_amount_tax_reductions_table_presenter.rb:57-62`:

```ruby
    ::DatabaseSelector.connected_to_reader do
      @records =
        Payroll::Infrastructure::QueryServices::IncomeTaxFixedAmountTaxReductionQueryService.execute(
          company_id: company.id,
        )
    end
```

Rules (`docs/knowledge/general/coding-standards/read-replica-usage.md`):

| ✅ Use it for | ❌ Don't use it for |
|---|---|
| Heavy, read-only queries (lists, reports, repository read methods) | Reading what you **just wrote** (replication lag → stale data) |
| | Inside a **transaction** (the replica can't see uncommitted writes) |
| | Anything that writes |

And **don't** call `ActiveRecord::Base.connected_to(role: :reading)` directly; go through `DatabaseSelector` (it handles the write fallback + notification).

---

## 8. Practice (15 min)

(`memos` / `Memo` is an **illustrative** model, not a real one in the repo.)

1. Rewrite correctly: `company.memos.where(deleted_at: nil)`
2. Soft-delete the record `memo`.
3. Soft-delete **all** memos of `employee` in one SQL.
4. This loops over 5,000 employees and prints their ids. Make it memory-friendly two ways (print ids; process full records):
   ```ruby
   company.employees.to_a.each { |e| puts e.id }
   ```
5. Fix the N+1: `company.employees.each { |e| puts e.company_reference_date_rule.name }`
6. You're writing a batch that must update records of **all** employees regardless of the logged-in user's department. What do you wrap it in, and what must you add?
7. `Attendance::CheckStatus` is in `checking`. Which event moves it to `check_finished`?

*(Answers:*
1. `company.memos.where(deleted_at: DEFAULT_DELETED_AT)` (or just `company.memos` if the model has the default_scope).
2. `memo.destroy_logically!` (or `memo.update!(deleted_at: Time.current)`).
3. `employee.memos.destroy_all_logically!`
4. `company.employees.pluck(:id).each { |id| puts id }` / `company.employees.find_each(batch_size: 1000) { |e| ... }`
5. `company.employees.preload(:company_reference_date_rule).each { ... }` (or `includes`).
6. `Authz.without_employee_leakcheck do ... end`, plus a **comment explaining why**.
7. `checked` (`cs.checked!`).)*

---

## 9. Summary

- **Soft delete**: `default_scope { where(deleted_at: DEFAULT_DELETED_AT) }` (88 models); delete with `destroy_logically!` / `destroy_all_logically!` (`config/initializers/logical_delete.rb`) or `update(deleted_at: Time.current)`. Never `deleted_at: nil`; `destroy` is physical.
- **Concerns** (`app/models/concerns/`): `extend ActiveSupport::Concern`, `included do` (class-level declarations), `class_methods do`.
- **Enums**: Rails `enum` (helpers `x?`, `x!`, scopes; `prefix: true`), `enumerize` gem (`predicates:`, `scope:`), **aasm** state machines (states, events, transitions).
- **LeakCheck** (from `ApplicationRecord`): records with an `employee_id` outside the accessible set raise. Use `.accessible` for lists; use `Authz.without_employee_leakcheck { }` + a reason comment when you truly need everyone.
- **Performance**: avoid N+1 with `includes`/`preload`/`eager_load`; for 1,000+ records use `pluck` / `find_each(batch_size: 1000)`, not `to_a`.
- **Transactions**: `ActiveRecord::Base.transaction do ... end`, all or nothing; raise to roll back.
- **Read replica**: `DatabaseSelector.connected_to_reader { }` for heavy read-only work; not after your own writes, not inside transactions.

➡️ Now take **Test13.md**.
