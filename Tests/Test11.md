# Test 11 — ActiveRecord 1: Models and Queries

**Rules:** Don't look at Lesson11.md. Write your answers under each question, then check `Answers11.md`.
**Total:** 44 points

Assume `company` is a `Company` record, and `Comment` has columns `id, company_id, user_id, url, page_name, comment, created_at, updated_at`, with `company.comments` available.

---

## Part A — Multiple choice (1 point each)

**1.** What is an ORM?
- a) A database server
- b) A library that lets you work with database rows as objects in your language
- c) A type of SQL join
- d) A JavaScript framework

**2.** `Comment.find(999)` when no such row exists:
- a) returns `nil`
- b) returns an empty array
- c) raises `ActiveRecord::RecordNotFound`
- d) creates the row

**3.** `Comment.find_by(url: '/nope')` when nothing matches:
- a) returns `nil`
- b) raises `RecordNotFound`
- c) returns `false`
- d) returns `[]`

**4.** What's the difference between `save` and `save!`?
- a) None
- b) `save!` raises an error if validation fails; `save` returns false
- c) `save!` skips validation
- d) `save` is faster

**5.** "Relations are lazy" means:
- a) Queries run slowly
- b) The SQL is not sent until the data is actually needed
- c) Rails caches every query forever
- d) Relations can only have one condition

**6.** `company.comments.pluck(:id)` returns:
- a) An array of `Comment` objects
- b) An array of plain id values like `[1, 2, 3]`
- c) A single integer
- d) A relation

**7.** In freee-payroll, `comment.destroy` does what?
- a) Soft delete (sets deleted_at)
- b) Physical delete (the row is removed)
- c) Nothing
- d) Archives to S3

**8.** Why use `where('datetime >= ?', date)` instead of `where("datetime >= '#{date}'")`?
- a) It's shorter
- b) The `?` placeholder escapes the value safely and prevents SQL injection
- c) `#{}` doesn't work in Ruby
- d) There is no difference

**9.** Which RuboCop cop flags `Employee.where(...)` called directly on the class?
- a) `Hr/UsedRailsLogger`
- b) `Freee/QueryFromActiveRecordClass`
- c) `Dependency/OverBoundary`
- d) `Style/StringLiterals`

---

## Part B — Fill in the blanks (1 point per blank)

**10.** All models in freee-payroll inherit from ______, which has `self.abstract_class = true`, meaning it has no ______ of its own.

**11.** CRUD stands for Create, ______, ______, Delete.

**12.** `Comment.new(...)` builds an object in memory; it is not saved until you call ______.

**13.** `comment.persisted?` returns ______ after a successful save.

---

## Part C — Match Ruby → SQL (1 point each)

Write the letter of the approximate SQL.

| # | Ruby | | | SQL |
|---|---|---|---|---|
| 14 | `company.comments.where(id: [1, 2])` | | A | `... ORDER BY created_at DESC LIMIT 5` |
| 15 | `company.comments.where.not(user_id: 5)` | | B | `SELECT COUNT(*) ... WHERE company_id = ?` |
| 16 | `company.comments.order(created_at: :desc).limit(5)` | | C | `... WHERE company_id = ? AND id IN (1, 2)` |
| 17 | `company.comments.count` | | D | `... WHERE company_id = ? AND user_id != 5` |

---

## Part D — Write the code (2 points each)

**18.** Find `company`'s comment with id `params_id`, raising if not found.

**19.** Get all of `company`'s comments by user 8 on url `'/employees'`, oldest first.

**20.** Get an array of the **urls** of all of `company`'s comments (just the strings).

**21.** Create a comment for `company` with user 3, url `'/home'`, comment `'OK'`, raising if invalid.

**22.** Update comment `c` so its `page_name` is `'ホーム'`, raising if invalid.

**23.** Return true if `company` has any comment by user 3.

---

## Part E — Fill in the missing code (1 point per blank)

**24.** Build the query step by step (only filter by `url` when it's given):

```ruby
def find_list(company_id:, url: nil)
  company = Company.____(company_id)
  scope = company.________
  scope = scope.where(url: url) ____ url.present?
  scope.order(created_at: :____)
end
```

---

## Part F — Answer in a sentence (2 points each)

**25.** Why does freee-payroll write `company.comments.find(id)` instead of `Comment.find(id)`? Give two reasons.

**26.** Why do freee-payroll repositories mostly use `create!` / `update!` instead of `create` / `update`?

**27.** A colleague writes `comment.destroy` for a model that has a `deleted_at` column. What might be wrong?

---

## Part G — Read real code (3 points)

`app/modules/payroll/infrastructure/repositories/company_daily_prorate_setting_repository.rb`:

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

**28.** Explain in 2–3 sentences what this method does, and what happens if `round_type` is invalid.
