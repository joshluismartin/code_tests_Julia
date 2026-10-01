# Lesson 10 — Databases, SQL and Migrations

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Understand how data is stored in MySQL, read basic SQL, and read migration files and `db/schema.rb` in freee-payroll.
**You need:** Lesson 1 (tables/rows/columns), Lesson 9. Open `~/Josh_Files/freee-payroll/db/` in your editor.

---

## 1. Relational databases: the vocabulary

| Term | Meaning | Example |
|---|---|---|
| **Database (DB)** | Organized, permanent storage for data. | freee-payroll's MySQL database |
| **DBMS** | The software that runs the database. | **MySQL** (in production: Amazon Aurora MySQL) |
| **Relational database** | Data is stored in **tables** that can **relate** to each other through ids. | `comments.company_id` → `companies.id` |
| **Table** | One kind of thing, like a spreadsheet sheet. | `comments` |
| **Row / record** | One item in the table. | One comment |
| **Column / field** | One attribute every row has. | `url`, `comment` |
| **Data type** | What kind of value a column holds. | `integer`, `bigint`, `string`, `text`, `datetime`, `date`, `boolean`, `decimal` |
| **Primary key (PK)** | A column whose value is unique per row, normally `id`. | `comments.id = 42` |
| **Foreign key (FK)** | A column holding *another table's* primary key. | `comments.company_id` |
| **NULL** | "No value at all" (≠ `0`, ≠ empty string `''`). | `page_name` may be NULL |
| **NOT NULL constraint** | Rule: this column must always have a value. | `null: false` |
| **Default** | The value used when none is given. | `default: "floor"` |
| **Index** | An extra lookup structure (like a book's index) that makes searching by certain columns fast. | index on `company_id` |
| **Unique index / constraint** | An index that also forbids duplicates. | `member_id` must be unique |
| **Schema** | The *shape* of the database: which tables and columns exist. | `db/schema.rb` |

### Data types in Rails vs Ruby

| Rails/MySQL column type | Ruby value you get |
|---|---|
| `integer`, `bigint` | `Integer` (`bigint` = bigger range) |
| `string` (short, ≤255 chars), `text` (long) | `String` |
| `boolean` | `true` / `false` |
| `date` | `Date` |
| `datetime` | `ActiveSupport::TimeWithZone` (a Time with time zone) |
| `decimal` | `BigDecimal` (exact decimal, good for money/rates) |
| NULL | `nil` |

---

## 2. SQL: the language of databases

| Term | Meaning |
|---|---|
| **SQL** (Structured Query Language) | The language used to ask the database for data or change it. |
| **Query** | One SQL command. |

You'll rarely write SQL by hand in freee-payroll (ActiveRecord writes it for you, Lesson 11), but you **must be able to read it**: it appears in logs, error messages, and some `where('...')` strings.

Imagine these two small tables (illustrative data):

`companies`

| id | name |
|---|---|
| 10 | 株式会社A |
| 20 | 株式会社B |

`comments`

| id | company_id | user_id | url | comment |
|---|---|---|---|---|
| 1 | 10 | 5 | /employees | 確認しました |
| 2 | 10 | 6 | /payroll | 金額を修正 |
| 3 | 20 | 9 | /employees | OK |

### 2.1 SELECT: read

```sql
SELECT * FROM comments;                          -- all columns, all rows
SELECT id, comment FROM comments;                -- only some columns
SELECT * FROM comments WHERE company_id = 10;    -- filter → rows 1 and 2
SELECT * FROM comments WHERE company_id = 10 AND user_id = 5;   -- → row 1
SELECT * FROM comments WHERE id IN (1, 3);       -- → rows 1 and 3
SELECT * FROM comments ORDER BY id DESC LIMIT 1; -- newest one → row 3
SELECT COUNT(*) FROM comments WHERE company_id = 10;  -- → 2
```

| Keyword | Meaning |
|---|---|
| `SELECT ... FROM table` | Which columns, from which table (`*` = all columns) |
| `WHERE` | Filter rows |
| `AND` / `OR` / `IN (...)` | Combine conditions / "is one of" |
| `IS NULL` / `IS NOT NULL` | Check for NULL (you can't use `= NULL`) |
| `ORDER BY col ASC/DESC` | Sort ascending/descending |
| `LIMIT n` | At most n rows |
| `COUNT(*)` | Count rows |

### 2.2 JOIN: combine tables

```sql
SELECT comments.comment, companies.name
FROM comments
INNER JOIN companies ON companies.id = comments.company_id
WHERE comments.user_id = 5;
-- → '確認しました', '株式会社A'
```

`JOIN` glues rows of two tables together where the `ON` condition matches (here: FK = PK).

### 2.3 INSERT / UPDATE / DELETE: write

```sql
INSERT INTO comments (company_id, user_id, url, comment) VALUES (10, 5, '/home', 'こんにちは');
UPDATE comments SET comment = '修正済み' WHERE id = 2;
DELETE FROM comments WHERE id = 3;
```

> ⚠️ `UPDATE` or `DELETE` **without** `WHERE` changes/deletes **every row**. Always check the `WHERE`.

### 2.4 Transactions (preview)

| Term | Meaning |
|---|---|
| **Transaction** | A group of SQL writes that succeed **all together** or fail **all together** (rollback). |
| **Commit** | Make the transaction's changes permanent. |
| **Rollback** | Undo everything in the transaction. |

Example: "save the new setting AND recalculate payroll": if the recalculation fails, the setting shouldn't be saved either. In Rails: `ActiveRecord::Base.transaction do ... end` (Lesson 13).

---

## 3. How Rails talks to MySQL

🔎 `config/database.yml:1-5` and `:15-32` (development; `...` = lines skipped):

```yaml
default: &default
  adapter:   mysql2
  charset:   utf8mb4
  encoding:  utf8mb4
  collation: utf8mb4_general_ci
  ...

development:
  primary: &development_primary
    <<: *default
    database:  <%= ENV['DB_NAME']||'freee_payroll_development' %>
    username:  <%= ENV['DB_USERNAME']||'mysql' %>
    password:  <%= ENV['DB_PASSWORD']||'root' %>
    host:      <%= ENV['DB_HOSTNAME']||'127.0.0.1' %>
    port:      <%= ENV['DB_PORT']||'23306' %>
    pool:      5
  primary_replica: &development_primary_replica
    ...
    replica:   true
```

| Term | Meaning |
|---|---|
| **adapter `mysql2`** | The gem Rails uses to talk to MySQL. |
| **utf8mb4** | Text encoding that supports all characters (Japanese, emoji). |
| **`<%= ENV['X'] \|\| 'default' %>`** | Read an **environment variable** (a setting from outside the code), else use a default. |
| **primary** | The main database: all **writes** go here. |
| **replica (read replica)** | A read-only **copy** of the primary, kept in sync automatically (with a tiny delay called *replication lag*). Used to spread the load of heavy reads. |

🔎 The model base class connects to both, in `app/models/application_record.rb:21-29`:

```ruby
  connects_to shards: {
                default: {
                  writing: :primary,
                  reading: :primary_replica,
                },
                write_forwarding: {
                  writing: :primary_replica_with_local_write_forwarding,
                },
              }
```

Normally everything uses `writing: :primary`. To read from the replica on purpose, freee-payroll wraps code in `DatabaseSelector.connected_to_reader do ... end` (Lesson 13).

---

## 4. Migrations: changing the database shape safely

| Term | Meaning |
|---|---|
| **Migration** | A Ruby file that describes **one change** to the database schema (add a table, add a column, add an index). |
| **Run / apply a migration** | Execute it against the DB: `bin/rails db:migrate`. |
| **Rollback** | Undo the last migration: `bin/rails db:rollback`. |
| **Version / timestamp** | The number at the start of the file name (`20260908000002` = 2026-09-08 00:00:02). Migrations run in this order. |
| **`schema_migrations` table** | A table where Rails records which versions have already run. |
| **DDL** | "Data Definition Language": SQL that changes structure (`CREATE TABLE`, `ALTER TABLE`). |

`db/migrate/` holds about **2,000** migration files. Every table and column ever added has one.

### 4.1 A real migration: create a table

🔎 `db/migrate/20260908000002_create_banner_recipient_companies.rb:1-12`:

```ruby
class CreateBannerRecipientCompanies < ActiveRecord::Migration[7.2]
  def change
    create_table :banner_recipient_companies, id: { type: :bigint, unsigned: true }, comment: 'バナーの表示対象事業所' do |t|
      t.bigint :banner_id, null: false, unsigned: true, comment: 'バナーID'
      t.bigint :company_id, null: false, unsigned: true, comment: '表示対象の事業所ID'
      t.timestamps

      t.index %i[banner_id company_id], unique: true
      t.index :company_id
    end
  end
end
```

Line by line:

| Code | Meaning |
|---|---|
| `< ActiveRecord::Migration[7.2]` | "This is a migration, written for Rails 7.2 rules." |
| `def change` | Rails can **automatically reverse** what's in `change` (for rollback). |
| `create_table :banner_recipient_companies` | Make a new table. |
| `id: { type: :bigint, unsigned: true }` | The `id` primary key is a big, non-negative integer. |
| `comment: '...'` | A description stored **in the DB** (freee-payroll writes these in Japanese). |
| `t.bigint :banner_id, null: false` | A required foreign-key column. |
| `t.timestamps` | Adds `created_at` and `updated_at` (Rails fills them automatically). |
| `t.index %i[banner_id company_id], unique: true` | A **composite unique index**: the same banner can't target the same company twice. |
| `t.index :company_id` | Fast lookup by company. |

### 4.2 A real migration: change an existing table

🔎 `db/migrate/20260907164253_add_egov_submission_id_to_employee_employment_insurance_transfer_notifications.rb:1-11`:

```ruby
class AddEgovSubmissionIdToEmployeeEmploymentInsuranceTransferNotifications < ActiveRecord::Migration[7.2]
  def change
    change_table :employee_employment_insurance_transfer_notifications, bulk: true do |t|
      t.bigint :egov_submission_id, unsigned: true, null: true, comment: 'reference id to egov submission table'
      # A filing links exactly one draft to one egov_submission (1:1; no bulk filing shares a submission),
      # so enforce the uniqueness at the DB layer. NULLs don't collide in MySQL, so the many unfiled
      # drafts (egov_submission_id NULL) coexist fine — only filed rows must be unique.
      t.index :egov_submission_id, name: 'idx_eeid_egov_submission_id', unique: true
    end
  end
end
```

`change_table ..., bulk: true` = combine several changes into **one** `ALTER TABLE` statement.

### 4.3 freee-payroll's migration rules

🔎 From `db/migrate/README.md` (written in Japanese; summarized):

1. **Use `def change`** when Rails can reverse it automatically.
2. **One DDL per file.** For multiple columns/indexes on one table, use `change_table :table, bulk: true`. (If a file has two DDLs and the second fails, retrying re-runs the first and crashes.)
3. **Irreversible operations** (`drop_table`, `remove_column`) go in `def up` / `def down`, and `down` must `raise ActiveRecord::IrreversibleMigration`.
4. Test `db:migrate` **and** `db:rollback` locally and check `db/schema.rb` has no unexpected diff.
5. Don't mix **NULL and empty string**: if NULL isn't needed, use `null: false`.
6. If something must be unique, add a **unique constraint in the DB** (not only in Ruby).
7. Discuss the table design with the team **before** opening a migration PR.

Also, the Dangerfile has `fail_on_migration_with_other_changes`: a PR with a migration should not contain other code changes.

---

## 5. `db/schema.rb`: the current shape of the database

| Term | Meaning |
|---|---|
| **`db/schema.rb`** | An auto-generated Ruby file describing **every table as it is now** (the result of all migrations). Don't edit it by hand; `db:migrate` updates it. |

🔎 `db/schema.rb:13`:

```ruby
ActiveRecord::Schema[7.2].define(version: 2026_09_08_000002) do
```

`version` = the latest migration applied. (It matches the banner migration above!)

🔎 The `comments` table, `db/schema.rb:711-721`:

```ruby
  create_table "comments", charset: "utf8mb4", collation: "utf8mb4_general_ci", options: "ENGINE=InnoDB ROW_FORMAT=DYNAMIC", force: :cascade do |t|
    t.bigint "company_id", null: false, comment: "事業所ID"
    t.bigint "user_id", null: false, comment: "ユーザーID"
    t.string "url", null: false, comment: "表示画面のURL"
    t.string "page_name", comment: "コメントに紐づく画面のパンくず名称"
    t.text "comment", null: false, comment: "コメント本体"
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
    t.index ["company_id", "user_id", "url"], name: "idx_company_id_user_id_url"
    t.index ["user_id"], name: "idx_user_id"
  end
```

Read it like a table definition: `comments` has `id` (implicit), `company_id`, `user_id`, `url`, `page_name` (nullable, no `null: false`), `comment`, timestamps, and two indexes.

🔎 The `employees` table, `db/schema.rb:5899-5915`:

```ruby
  create_table "employees", id: :integer, charset: "utf8mb4", collation: "utf8mb4_general_ci", comment: "従業員情報", options: "ENGINE=InnoDB ROW_FORMAT=DYNAMIC", force: :cascade do |t|
    t.integer "company_id", null: false
    t.bigint "company_reference_date_rule_id", comment: "対象の締め日・支払日セット"
    t.string "base_pension_num", comment: "基礎年金番号"
    t.string "num", comment: "従業員番号"
    t.string "display_name", default: "", null: false, comment: "表示名"
    t.bigint "person_id", comment: "マイナンバーDBのPerson ID"
    t.datetime "deleted_at", precision: nil, default: "1000-01-01 00:00:00", null: false
    t.datetime "created_at", precision: nil
    t.datetime "updated_at", precision: nil
    t.string "employment_insurance_reference_number", limit: 11, default: "", null: false, comment: "被保険者番号（雇用保険）"
    t.string "member_id", limit: 26
    t.index ["company_id", "person_id"], name: "employees_i1"
    t.index ["company_reference_date_rule_id"], name: "index_employees_on_company_reference_date_rule_id"
    t.index ["member_id"], name: "index_employees_on_member_id", unique: true
    t.index ["person_id"], name: "index_employees_on_person_id"
  end
```

Two surprising things to notice:

1. **The employees table is thin.** Name, address, salary... are *not* here. They live in many `employee_*_rules` tables (e.g. `employee_profile_rules`, `employee_basic_pay_rules`), because HR data changes over time and each rule table stores values with validity periods.
2. **`deleted_at` defaults to `"1000-01-01 00:00:00"` and is `null: false`.** That's freee-payroll's **soft delete** convention (next section).

---

## 6. Soft delete (論理削除): a freee-payroll rule you MUST know

| Term | Meaning |
|---|---|
| **Hard / physical delete (物理削除)** | The row is really removed with SQL `DELETE`. Gone forever. |
| **Soft / logical delete (論理削除)** | The row stays, but a column marks it as deleted. Useful for history, audits, and undo. |

In many Rails apps, soft delete means "`deleted_at` is NULL = alive, has a time = deleted". **freee-payroll is different:**

🔎 `config/constants.rb:51`:

```ruby
DEFAULT_DELETED_AT = Time.utc(1000, 1, 1)
```

| `deleted_at` value | Meaning in freee-payroll |
|---|---|
| `1000-01-01 00:00:00` (= `DEFAULT_DELETED_AT`) | **Not deleted** (alive) |
| any real time, e.g. `2026-09-30 10:00:00` | **Deleted** at that time |
| `NULL` | ❌ Never happens: the column is `null: false` |

So in SQL, "alive rows" are:

```sql
SELECT * FROM employees WHERE deleted_at = '1000-01-01 00:00:00';
```

and **`WHERE deleted_at IS NULL` is always wrong** in this repo (it matches nothing). `app/README.md` says it directly: *"deleted_at: nil / deleted_at IS NULL の条件は常に誤り"* ("conditions on deleted_at nil / IS NULL are always wrong").

> Why year 1000? The docs don't say, but a likely reason: it keeps `deleted_at` `NOT NULL`, so uniqueness rules can include `deleted_at` (MySQL never treats two NULLs as duplicates). 🔎 For example `app/models/user_employee.rb:21` has `validates :employee_id, uniqueness: { scope: %i[company_id deleted_at] }`, meaning only one *alive* row per employee and company.

You'll see how models use this in Lesson 13.

---

## 7. From Ruby to SQL (preview of Lesson 11)

ActiveRecord turns Ruby method calls into SQL. The exact SQL may differ slightly; this is *approximately* what gets sent:

| Ruby | SQL (approximate) |
|---|---|
| `Comment.find(1)` | `SELECT comments.* FROM comments WHERE comments.id = 1 LIMIT 1` |
| `company.comments` | `SELECT comments.* FROM comments WHERE comments.company_id = 10` |
| `company.comments.where(url: '/payroll')` | `... WHERE company_id = 10 AND url = '/payroll'` |
| `company.comments.count` | `SELECT COUNT(*) FROM comments WHERE company_id = 10` |
| `company.comments.order(id: :desc).limit(1)` | `... ORDER BY id DESC LIMIT 1` |
| `company.employees` | `... FROM employees WHERE company_id = 10 AND deleted_at = '1000-01-01 00:00:00' AND company_reference_date_rule_id IS NOT NULL` (default scopes, Lesson 12–13) |

In development, you can see the real SQL in `log/development.log` or in the console output.

---

## 8. Practice (15 min)

**Practice 1: read SQL.** Using the `comments` table in §2, what does each return?

1. `SELECT id FROM comments WHERE url = '/employees';`
2. `SELECT COUNT(*) FROM comments WHERE company_id = 20;`
3. `SELECT comment FROM comments WHERE company_id = 10 ORDER BY id DESC LIMIT 1;`

**Practice 2: read schema.** Open `db/schema.rb`, search `create_table "company_daily_prorate_settings"`. Answer: what is the default of `round_type`? Is `company_id` required? What does the comment say?

**Practice 3: write a migration (on paper).** Add a nullable `string` column `nickname` with comment `'呼び名'` and an index on it to the `comments` table, following freee-payroll's rule (one DDL, `bulk: true`).

*(Answers: 1 → ids 1 and 3. 2 → 1. 3 → '金額を修正'. Practice 2: `default: "floor"`, `company_id` is `null: false` (required), comment "日割り計算の端数処理方法. floor: 切り捨て, round: 四捨五入, ceil: 切り上げ". Practice 3:)*

```ruby
class AddNicknameToComments < ActiveRecord::Migration[7.2]
  def change
    change_table :comments, bulk: true do |t|
      t.string :nickname, comment: '呼び名'
      t.index :nickname
    end
  end
end
```

---

## 9. Summary

- A **relational DB** stores data in **tables** (rows × columns) linked by **primary keys** and **foreign keys**.
- **SQL**: `SELECT ... FROM ... WHERE ... ORDER BY ... LIMIT`, `JOIN`, `INSERT`, `UPDATE`, `DELETE`. Beware `UPDATE`/`DELETE` without `WHERE`.
- freee-payroll uses **MySQL** via `mysql2`, with a **primary** (writes) and a **read replica** (reads).
- **Migrations** in `db/migrate/` change the schema; freee rules: `def change`, one DDL per file (`change_table ..., bulk: true`), DB-level unique constraints, `null: false` when NULL isn't needed.
- **`db/schema.rb`** shows the current shape; it's generated, not hand-edited.
- **Soft delete:** `deleted_at = '1000-01-01'` (`DEFAULT_DELETED_AT`) means **alive**. `deleted_at IS NULL` is **always wrong** here.

➡️ Now take **Test10.md**.
