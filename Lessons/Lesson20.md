# Lesson 20 — RSpec Basics

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Be able to open any `_spec.rb` file in `freee-payroll/spec/` and say, in plain words, *what is being tested* and *what result is expected*.
**You need:** Lessons 3–19 (Ruby, blocks, classes, Rails models, services). Open `~/Josh_Files/freee-payroll/spec/` in your editor.

---

## 1. What is an automated test?

| Term | Meaning in plain words | Example |
|---|---|---|
| **Test** | Code that runs *your* code and checks the result is what you expected. | "When I call `age_at(date)`, I expect 20." |
| **Automated test** | A test a computer runs for you, again and again, in seconds. No clicking in the browser. | `bin/rspec spec/models/bank_spec.rb` |
| **Test suite** | All the tests of a project together. freee-payroll's suite has tens of thousands of tests. | Everything in `spec/` |
| **Pass / Fail** | *Pass* = the result matched the expectation (green). *Fail* = it didn't (red). | `1 example, 0 failures` |
| **Regression** | A bug where something that *used to work* breaks because of a new change. Tests catch these. | You fix bonus calc, salary calc breaks. |
| **RSpec** | The Ruby testing tool (a gem) freee-payroll uses. "Spec" is short for *specification*: a description of how code should behave. | `gem 'rspec-rails'` in `Gemfile` |
| **Spec file** | A Ruby file containing tests. Its name always ends in `_spec.rb`. | `comment_spec.rb` |
| **CI** | Continuous Integration: a server (CircleCI here) that runs the whole suite on every pull request. A red test blocks merging. | `.circleci/config.yml` job `rspec` |

**Why bother?** freee-payroll calculates people's salaries and taxes. A wrong number is a real problem for a real person. Tests are the safety net that lets hundreds of developers change code every day without breaking each other's work.

### 1.1 Kinds of tests you will see

| Kind | What it tests | In freee-payroll |
|---|---|---|
| **Unit test** | One small piece (one class or method) in isolation. | `spec/models/`, `spec/services/`, `spec/values/`, `spec/modules/` |
| **Request spec** (integration test) | A whole HTTP request going through routes → controller → service → DB → JSON response. | `spec/controllers/api/private/**` with `type: :request` (Lesson 21) |
| **Controller spec** | Only a controller in isolation. **Restricted** in freee-payroll to base classes and concerns. | Lesson 21 |

---

## 2. Where spec files live: the mirror rule

A spec file lives at the **same path as the code it tests**, with `app/` replaced by `spec/` and `.rb` replaced by `_spec.rb`:

| Code | Spec |
|---|---|
| `app/models/approval_request/approver_access_control.rb` | `spec/models/approval_request/approver_access_control_spec.rb` |
| `app/services/application/payroll_verifications/upload_file_service.rb` | `spec/services/application/payroll_verifications/upload_file_service_spec.rb` |
| `lib/freee_sign_client/revoke_token_client.rb` | `spec/lib/freee_sign_client/revoke_token_client_spec.rb` |

This is not only a habit: a CI bot called **Danger** checks it. In `Dangerfile` the method `warn_on_not_correspond_impl_to_spec` (defined at `Dangerfile:5`) computes the expected spec path and warns on your pull request if it's missing:

```ruby
# Dangerfile:21-23
        elsif file.start_with?('app/')
          [file.gsub(%r{\Aapp\/}, 'spec/').gsub(/\.rb/, '_spec.rb')]
        elsif file.start_with?('lib/')
```

> Read it: "if the changed file starts with `app/`, the expected spec is the same path with `app/` → `spec/` and `.rb` → `_spec.rb`."

**Tip for reading the repo:** found a class and want to see how it's *supposed* to behave? Open its mirrored spec. Tests are often the best documentation.

---

## 3. The skeleton of a spec file

Here is a real, complete spec. We'll use it all lesson.

```ruby
# spec/models/approval_request/approver_access_control_spec.rb:1-20
require 'rails_helper'

RSpec.describe ApprovalRequest::ApproverAccessControl do
  let(:company) { default_company }
  let(:approver) { create(:employee, company_id: company.id) }
  let(:applicant) { create(:employee, company_id: company.id) }
  let(:other_company) { create(:company) }

  describe 'validate' do
    subject { build(:approver_access_control, approver: approver, applicant: applicant) }

    context 'ok' do
      it { expect(subject).to be_valid }
    end

    context '異なる事業所IDの従業員がセットされた' do
      let(:applicant) { create(:employee, company_id: other_company.id) }
      it { expect { subject.valid? }.to raise_error(RuntimeError) }
    end
  end
```

### 3.1 `require 'rails_helper'`

The first line loads `spec/rails_helper.rb`, which boots Rails in the **test environment** (a separate test database), loads helpers like `default_company`, FactoryBot, time helpers, and global stubs. **Every spec in freee-payroll starts with this line.**

### 3.2 `describe`, `context`, `it`: the three building blocks

| Word | Meaning | Convention |
|---|---|---|
| `RSpec.describe X do` / `describe X do` | "These tests are about **X**." The top level names a class. (In freee-payroll both `RSpec.describe` and bare `describe` appear; they're the same.) | Top: the class. Inside: a method name, like `'#execute'` (instance method) or `'.execute'` (class method). |
| `context '...' do` | "**In this situation**…" Groups tests that share a condition. | Starts with *when / with / if*, or a Japanese condition like `'異なる事業所IDの従業員がセットされた'` ("employees from different companies were set"). |
| `it '...' do` | **One test** (also called an **example**). Contains the expectations. | Describes the expected behavior. May have no description: `it { ... }`. |

They are just **Ruby methods that take a block** (Lesson 5), so they nest:

```
describe ApprovalRequest::ApproverAccessControl      ← the class
  describe 'validate'                                ← the behavior
    context 'ok'                                     ← situation 1
      it { expect(subject).to be_valid }             ← test 1
    context '異なる事業所IDの従業員がセットされた'           ← situation 2
      it { expect { subject.valid? }.to raise_error(RuntimeError) }   ← test 2
```

When RSpec prints a failure, it glues the names together: *"ApprovalRequest::ApproverAccessControl validate ok is expected to be valid"*. Good names = readable failures.

**`#` vs `.` in describe names**: `describe '#get_access_control_to'` means an **instance** method (`employee.get_access_control_to`); `describe '.execute'` means a **class** method (`Service.execute`). This is a Ruby-documentation convention.

---

## 4. Expectations: `expect(...).to ...`

An **expectation** (or **assertion**) is the actual check.

```ruby
expect(actual_value).to matcher(expected_value)
expect(actual_value).not_to matcher(expected_value)
```

A **matcher** is the word that says *how* to compare. The most common ones:

| Matcher | Passes when… | Real / illustrative example |
|---|---|---|
| `eq(x)` | value `==` x | `expect(subject.approver).to eq(approver)` (`approver_access_control_spec.rb:28`) |
| `be(x)` | *the very same object* (identity). Used with `true`/`false`/`nil`. | `expect(subject.expired?).to be(false)` (`:31`) |
| `be true` / `be false` | same as `be(true)` (parentheses optional) | `expect(subject).to be true` (`:49`) |
| `be_truthy` / `be_falsey` | anything truthy / falsy (Lesson 4: only `nil` and `false` are falsy) | `expect(result).to be_truthy` (illustrative) |
| `be_nil` | value is `nil` | `expect(employee.retire_date).to be_nil` (illustrative) |
| `be_valid` | calls `.valid?` and expects `true` (this is a *predicate matcher*: `be_xxx` calls `xxx?`) | `expect(subject).to be_valid` (`:13`) |
| `include(x)` | array/string/hash contains x | `expect(%w[a b]).to include('a')` (illustrative) |
| `match(x)` | matches a regex, or a nested structure (with `hash_including` etc.) | `expect('EMP001').to match(/\AEMP/)` (illustrative) |
| `have_attributes(a: 1)` | object's methods return those values | `expect(res).to have_attributes(file_name: 'sample.jpg', ...)` (`spec/modules/employee_master/infrastructure/repositories/zenshoku_gensenchoshu_hyo_repository_spec.rb:44`) |
| `contain_exactly(a, b)` | array has exactly these elements, **any order** | `expect(violated_classes).to contain_exactly(*ignore_list)` (`spec/service_class_policies_spec.rb:417`) |

**`eq` vs `be`** — beginners mix these up:
- `eq` compares **values**: `expect('abc').to eq('abc')` passes.
- `be` compares **identity** (same object in memory). For `true`, `false`, `nil`, and small integers there is only one object, so `be` is safe there. Use `eq` for everything else.

### 4.1 Block expectations: `expect { ... }`

Some matchers need to **run code** and watch what happens *during* it. For those you pass a **block** `{ }` instead of a value `( )`:

**`change`** — did running the block change something?

```ruby
# approver_access_control_spec.rb:27
expect { subject }.to change(ApprovalRequest::ApproverAccessControl, :count).by(1)
```
Read it: *"Running `subject` increases `ApprovalRequest::ApproverAccessControl.count` by 1"* → i.e. it creates one record.

There are two forms:

| Form | How it reads the value |
|---|---|
| `change(Klass, :count)` | calls `Klass.count` before and after |
| `change { any_ruby_expression }` | runs the inner block before and after |

```ruby
# spec/services/application/payroll_verifications/upload_file_service_spec.rb:32
expect { subject }.to change { company.payroll_comparison_imports.count }.by(1)
```

`.by(-1)` means "decreased by 1" (a delete). You can also use `.from(x).to(y)`.

**`raise_error`** — did the block raise an exception (Lesson 7)?

```ruby
# upload_file_service_spec.rb:52
expect { subject }.to raise_error(::HrErrors::UserError, 'ファイルサイズは3 MB以内にしてください。')
```
Read it: *"Running `subject` raises `HrErrors::UserError` with exactly that message"* ("file size must be within 3 MB").

And the opposite: `expect { subject }.not_to raise_error` (`spec/lib/freee_sign_client/revoke_token_client_spec.rb:28`).

> ⚠️ **Parentheses vs braces matter.** `expect(subject).to raise_error` is **wrong**: `subject` is evaluated *before* `expect` can catch anything, so the error escapes and the test crashes. Error and change checks always need `expect { }`.

### 4.2 `not_to`

Flips any matcher:

```ruby
# spec/models/approval_request/comment_spec.rb:73-75
context 'nilはNG' do
  let(:speaker) { nil }
  it { expect(subject).not_to be_valid }
end
```
("nil is NG" → a comment whose speaker is `nil` must be **invalid**.)

---

## 5. Setting up data: `let`, `let!`, `subject`, `before`

### 5.1 `let`: a named, lazy helper value

```ruby
let(:company) { default_company }
let(:approver) { create(:employee, company_id: company.id) }
```

- `let(:name) { value }` defines a method `name` you can call inside tests.
- **Lazy**: the block runs only the **first time** you call `name` inside a test, and the result is **memoized** (remembered; Lesson 4's `||=` idea) for the rest of that test.
- Each `it` gets a **fresh** value. Tests never share `let` values.
- An inner `context` can **override** a `let`. See `approver_access_control_spec.rb:17`: inside that context, `applicant` is replaced by an employee from `other_company`, and `subject` (which uses `applicant`) automatically uses the new one. This is how one `subject` serves many situations.

### 5.2 `let!`: eager

`let!` (with a bang) runs **before every test**, even if you never call it. Use it when the record must *exist in the database* even though the test doesn't mention it by name.

```ruby
# spec/controllers/api/private/shift/shift_requests_controller_spec.rb:5
let!(:special_holiday_setting) { create(:special_holiday_setting, company: company, name: '結婚休暇', usage_day: 1.0) }
```
The controller will look up special holidays from the DB, so the row must be there beforehand. With plain `let`, it would never be created.

| | `let` | `let!` |
|---|---|---|
| When it runs | First time it's called | Before each example, always |
| Use for | Values the test uses directly | Records that must pre-exist in the DB |

### 5.3 `subject`: "the thing under test"

```ruby
# approver_access_control_spec.rb:22-23
describe '#get_access_control_to' do
  subject { approver.get_access_control_to applicant }
```

- `subject { ... }` is a special `let` named `subject`: it's lazy and memoized.
- In freee-payroll, `subject` is usually **the call being tested** (a method call, or an HTTP request).
- Because it's lazy, `expect { subject }` is what *actually runs* the code. Inside `change`/`raise_error` blocks this timing matters.

**`is_expected`** is shorthand for `expect(subject)`:

```ruby
it { is_expected.to be true }        # same as: it { expect(subject).to be true }
```
You'll see `is_expected.to eq 200` everywhere in request specs.

### 5.4 `described_class`

Inside `describe SomeClass do`, `described_class` returns `SomeClass`. It saves typing long names and survives renames:

```ruby
# upload_file_service_spec.rb:5, 17
describe Application::PayrollVerifications::UploadFileService, type: :service do
  ...
    subject { described_class.execute(company: company, user_id: 1, year: year, month: month, file: file) }
```
`described_class.execute(...)` = `Application::PayrollVerifications::UploadFileService.execute(...)`. (Remember Lesson 17: services have one public class method, `execute`.)

### 5.5 `before`: run setup code

```ruby
# approver_access_control_spec.rb:43-46
before do
  travel_to(Time.zone.now)
  approver.get_access_control_to applicant
end
```
- `before do ... end` runs **before every `it`** in its group (and nested groups).
- Use it for **actions** (create an access control, log in, stub something). Use `let` for **values**.
- Outer `before` blocks run before inner ones.

### 5.6 Putting it together: the order things happen

For each `it`:
1. All `let!` and `before` blocks run, from the outermost group inward.
2. The `it` body runs. Any `let`/`subject` is computed the first time it's touched.
3. The database is **rolled back** (`config.use_transactional_fixtures = true`, `spec/rails_helper.rb:394`), so the next test starts clean.

---

## 6. Reading a whole spec, step by step

```ruby
# spec/models/approval_request/approver_access_control_spec.rb:40-70
describe '#has_access_control_to?' do
  subject { approver.has_access_control_to? applicant }

  before do
    travel_to(Time.zone.now)
    approver.get_access_control_to applicant
  end

  context '正常系' do
    it { expect(subject).to be true }
  end

  context 'アクセス猶予による制御' do
    it '時間内はアクセス可能' do
      travel_to(Time.zone.now.since(ApprovalRequest::ApproverAccessControl::EXPIRE_MINUTES.minutes))
      expect(subject).to be true
    end

    it '時間外はアクセス不能' do
      travel_to(Time.zone.now.since(ApprovalRequest::ApproverAccessControl::EXPIRE_MINUTES.minutes + 1))
      expect(subject).to be false
    end
  end

  context 'アクセス権限がない' do
    subject { employee.has_access_control_to? applicant }

    let(:employee) { create(:employee, company_id: company.id) }
    it { expect(subject).to be false }
  end
end
```

In plain English:

- **What's tested:** `Employee#has_access_control_to?(applicant)` (defined at `app/models/employee.rb:1630`): "can this approver see this applicant's data?"
- **Setup (`before`):** freeze the clock (`travel_to`, Lesson 21) and grant the approver access to the applicant.
- **正常系** ("normal case"): right after granting, the answer is `true`.
- **アクセス猶予による制御** ("control by access grace period"). `EXPIRE_MINUTES` is `60` (`app/models/approval_request/approver_access_control.rb:2`):
  - move the clock forward exactly 60 minutes → still `true` ("accessible within time")
  - 60 minutes **+ 1 second** → `false` ("not accessible after time")
- **アクセス権限がない** ("has no access"): a *different* employee who was never granted access → `false`. Note how the inner context **overrides `subject`**.

Common Japanese words in spec names:

| Japanese | Meaning |
|---|---|
| 正常系 (seijōkei) | normal / happy path |
| 異常系 (ijōkei) | error path |
| 〜の場合 | "in the case of ~" |
| 〜こと | "(it) should ~" (e.g. 削除されること = "should be deleted") |
| NG / OK | invalid / valid |

---

## 7. Loops inside specs

Because specs are plain Ruby, you can generate contexts with `each`:

```ruby
# spec/models/approval_request/comment_spec.rb:53-79
describe 'speaker' do
  subject { approval_request.comments.build({ body: body, speaker: speaker }) }

  let(:body) { 'これはコメントです.' }
  let(:approval_request) do
    create(:payroll_approval_request, company: company, employee: employee, requested_on: requested_on, date: date)
  end

  context '正常系' do
    %i[approver applicant payroll admin bot proxy_applicant].each do |speaker_type|
      context "#{speaker_type}の場合" do
        let(:speaker) { speaker_type }
        it { expect(subject).to be_valid }
      end
    end
  end

  context 'nilはNG' do
    let(:speaker) { nil }
    it { expect(subject).not_to be_valid }
  end

  context '範囲外はNG' do
    let(:speaker) { :dummy }
    it { expect { subject }.to raise_error(ArgumentError) }
  end
end
```

- `%i[approver applicant ...]` is an array of symbols (Lesson 4).
- For each allowed speaker type, a new context is generated that expects the comment to be valid. That's 6 tests from 5 lines.
- `nil` → invalid; an unknown value `:dummy` → `ArgumentError` (the enum rejects it).

Lesson 21 teaches the team's official style for this, **Data Driven Tests**.

---

## 8. Running specs

From the repo root (you need the dev environment running; ask your team how to set it up):

```bash
bin/rspec spec/models/approval_request/approver_access_control_spec.rb      # whole file
bin/rspec spec/models/approval_request/approver_access_control_spec.rb:53   # only the example/group on line 53
bin/rspec spec/models/approval_request/                                      # whole folder
```

`bin/rspec` is a small script in `bin/` that runs RSpec with the project's settings.

Reading output:

```
..F.

Failures:
  1) ApprovalRequest::ApproverAccessControl#has_access_control_to? アクセス猶予による制御 時間外はアクセス不能
     Failure/Error: expect(subject).to be false
       expected false
            got true
     # ./spec/models/approval_request/approver_access_control_spec.rb:60

4 examples, 1 failure
```
- `.` = passed, `F` = failed.
- The failure shows the **full name** (describe + context + it), the **line**, and **expected vs got**.

*(Output above is illustrative.)*

---

## 9. Practice (15 min)

**Practice 1 — label it.** Open `spec/services/application/payroll_verifications/upload_file_service_spec.rb`. For lines 1–54, write next to each line one of: *setup*, *subject*, *group*, *test*, *expectation*.

**Practice 2 — translate.** Write each line in plain English:
1. `expect { subject }.to change(Comment, :count).by(-1)`
2. `it { is_expected.to eq 200 }`
3. `expect(result).to eq({ id: import.id, csv_headers: %w[氏名 支給額] })`
4. `expect { subject }.not_to raise_error`
5. `let!(:cya) { create(:company_yearend_adjustment, company: company, year: year) }`

**Practice 3 — spot the bug.** Why does this test crash instead of passing?
```ruby
it { expect(subject).to raise_error(HrErrors::UserError) }
```

**Practice 4 — find the mirror.** What is the spec path for `app/modules/payroll/infrastructure/repositories/company_allowance_repository.rb`? Check with `ls` whether it exists.

<details><summary>Practice answers</summary>

1. Practice 1 sample: line 3 = setup (loads Rails); 5 = group; 6–8 = setup (`let`); 10–14 = setup (`before` stub); 16 = group; 17 = subject; 19 = group (context); 20–28 = setup; 30 = test; 32/35/38 = expectations; 41 = test; 42–44 = expectation; 48 = group; 49 = setup; 51 = test; 52 = expectation.
2. (1) Running subject deletes one Comment. (2) The subject (an HTTP request) returns status 200. (3) The result equals a hash with the import's id and the CSV headers ["氏名", "支給額"]. (4) Running subject doesn't raise any error. (5) Before every test, create a company year-end adjustment record for this company and year.
3. `expect(subject)` evaluates `subject` immediately, so the error is raised before RSpec can catch it. Use `expect { subject }.to raise_error(...)`.
4. `spec/modules/payroll/infrastructure/repositories/company_allowance_repository_spec.rb`.
</details>

---

## 10. Summary

- A **spec** describes expected behavior; **RSpec** runs it; **CI** runs all of them on every PR.
- Spec path **mirrors** the code path (`app/x/y.rb` → `spec/x/y_spec.rb`); Danger warns if missing.
- `describe` (what) → `context` (situation) → `it` (one example) → `expect(...).to matcher`.
- Matchers: `eq`, `be`, `be_valid`, `include`, `match`, `have_attributes`, `contain_exactly`; block matchers `change` and `raise_error` need `expect { }`.
- `let` = lazy + memoized; `let!` = eager; `subject` = the thing under test; `is_expected` = `expect(subject)`; `described_class` = the class in `describe`; `before` = setup actions.
- Run with `bin/rspec path_spec.rb:LINE`.

➡️ Now take **Test20.md**.
