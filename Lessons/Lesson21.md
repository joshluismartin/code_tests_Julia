# Lesson 21 — RSpec in freee-payroll

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Read (and start writing) specs the way the freee-payroll team does: factories, `default_company`, request specs, mocks/stubs, time travel, Data Driven Tests, and the team's testing rules.
**You need:** Lesson 20 (describe/context/it, `let`, `subject`, matchers). Open `~/Josh_Files/freee-payroll/spec/` in your editor, and also `spec/CLAUDE.md` / `spec/README.md` and `spec/controllers/README.md`.

---

## 1. New words for this lesson

| Term | Meaning in plain words |
|---|---|
| **Test data** | Records you create just for a test (an employee, a comment...). |
| **Factory** | A recipe that builds test records with sensible default values, so you only write the fields you care about. |
| **FactoryBot** | The gem that provides factories (`gem 'factory_bot_rails'`). |
| **Seed data** | Data loaded into the test DB once, before all tests (tax tables, and one Company). |
| **Test double** | A fake object that stands in for a real one (like a stunt double in a film). |
| **Stub** | Telling an object "when someone calls method X, just return Y", so the real method doesn't run. |
| **Mock / message expectation** | A stub that also *checks* the method was called (with the right arguments). |
| **Spy** | Stub first, then check afterwards with `have_received`. |
| **WebMock** | A gem that blocks real HTTP calls in tests and lets you fake responses (`stub_request`). |
| **Flaky test** | A test that passes sometimes and fails other times without any code change. Very annoying; the team has rules to prevent it. |
| **Shared examples** | A reusable group of tests you can plug into many specs. |

---

## 2. The team's rules (`spec/CLAUDE.md` → `spec/README.md`)

Memorize these. Reviewers will point them out.

| Rule (from `spec/README.md`) | What it means |
|---|---|
| Don't specify `id` when creating records with FactoryBot. | Hard-coded ids collide → flaky tests. |
| Write parameterized tests in the **Data Driven Test** pattern. | See §7. |
| Don't specify `aggregate_failures`; it's on globally. | See §2.1. |
| Tests under the same condition go into **one example**, not split. | One `it` can have several `expect`s. |
| Don't write tests that just **transcribe the implementation** to raise coverage. | Test *behavior*, not "line 3 calls method X". |
| Don't use custom matchers (if unavoidable, put them in `spec/support/matchers` with the same file name). | Stick to standard RSpec matchers. |
| Use `default_company`. `create(:company)` / `build(:company)` is basically forbidden unless you truly need **another** company. | Creating a company is very slow (lots of `after_create` work). |

### 2.1 How the rules are enforced

**`aggregate_failures` is on everywhere:**
```ruby
# spec/rails_helper.rb:82
config.define_derived_metadata { |meta| meta[:aggregate_failures] = true }
```
Normally RSpec stops an example at the **first** failed `expect`. With *aggregate failures*, it runs **all** `expect`s in the example and reports every failure at once. That's why the team happily writes several `expect`s in one `it`.

**No explicit ids:**
```ruby
# spec/factories.rb:1-9
FactoryBot.define do
  before(:create) do |object|
    if object.id && object.is_a?(ActiveRecord::Base) &&
         # 一括更新やインポートのテストでは、spec/fixtures 以下のデータを使用しているため、id 指定を許容する
         %w[import mass_update].none? { |path| RSpec.current_example.file_path.include?(path) }
      raise "Avoid specifying the id (#{object.id}) in the factory for #{object.class.name} to prevent flaky tests."
    end
  end
end
```
Read it: *"Before creating any record, if an id was given, raise an error, unless the spec file's path contains `import` or `mass_update`"* (the comment explains why: those tests use fixture CSV files under `spec/fixtures` that reference fixed ids).

---

## 3. `default_company`

```ruby
# spec/rails_helper.rb:64-77
config.include(
  Module.new do
    # Retrieve `Company` for standard tests from pre-seeded data.
    # ...
    def default_company
      return @default_company if defined?(@default_company)
      @default_company = Company.first
    end
  end,
)
```

- Before the whole suite runs, the test DB is seeded **once** and **one** `Company` is created.
- `default_company` just returns that company (`Company.first`), memoized.
- Almost every spec starts with `let(:company) { default_company }`.

When *is* `create(:company)` OK? When the test is literally about "a different company". Real example:

```ruby
# spec/models/approval_request/approver_access_control_spec.rb:7, 16-19
let(:other_company) { create(:company) }
...
context '異なる事業所IDの従業員がセットされた' do
  let(:applicant) { create(:employee, company_id: other_company.id) }
  it { expect { subject.valid? }.to raise_error(RuntimeError) }
end
```
The point of this test is "the approver and applicant belong to *different* companies", so a second company is unavoidable. And because it's a lazy `let`, the slow company is only created in that one context.

---

## 4. FactoryBot

### 4.1 Defining factories (`spec/factories/*.rb`, ~390 files)

```ruby
# spec/factories/company_profile_custom_fields.rb:1-13
FactoryBot.define do
  factory :company_profile_custom_field do
    sequence(:name) { |n| "test#{n}" }
    field_type { :text }
    open_status { :draft }
    is_reset_every_month { true }

    trait :number do
      field_type { :number }
      open_status { :published }
    end
  end
end
```

| Piece | Meaning |
|---|---|
| `factory :company_profile_custom_field do` | Recipe for the `CompanyProfileCustomField` model (name → class by Rails convention). |
| `field_type { :text }` | Default value. Always written as a **block** `{ }` so it's computed fresh each time. |
| `sequence(:name) { \|n\| "test#{n}" }` | Generates unique values: `test1`, `test2`, ... Good for columns that must be unique. |
| `trait :number do ... end` | A named **variation** that overrides some defaults. Use it as `create(:company_profile_custom_field, :number)`. |

More factory features you'll see:

```ruby
# spec/factories/employee_hr_portal_sub_actions.rb:4-8
factory :employee_hr_portal_sub_action do
  employee { association :employee, company: company }   # build an associated record, passing options
  hr_portal_sub_action                                   # implicit association: uses the :hr_portal_sub_action factory
  note_plaintext { '実施メモ' }
  status { 'todo' }
```

```ruby
# spec/factories/hr_portal_actions.rb:9-11
trait :with_sub_actions do
  after(:create) { |action| create_list(:hr_portal_sub_action, 2, hr_portal_action: action) }
end
```
`after(:create)` is a **callback**: after the record is saved, run this block (here: create 2 sub-actions for it).

```ruby
# spec/factories/elearning_users.rb:2
factory :elearning_user, class: 'Elearning::User' do
```
`class:` is needed when the factory name doesn't match the class name (namespaced models).

The most important factory is **`:employee`**:

```ruby
# spec/factories/employees.rb:1-15 (excerpt)
FactoryBot.define do
  factory :employee do
    member_id { ULID.generate }

    # company_reference_date_rule_idが指定されていないが、
    # companyだけ指定されているときはcompany_reference_date_rule_idを自動でセットしておく
    company_reference_date_rule do |employee|
      company = employee.company || (Company.find(employee.company_id) if employee.company_id)
      company.try(:company_reference_date_rules).try(:first)
    end

    trait :with_rules do
      after(:create) do |employee|
        # ::EmployeeMaster::CreateEmployeeRulesForPayrollService#execute! 参照
        create(:employee_profile_rule, company: employee.company, employee: employee)
        ...
```
- It sets `company_reference_date_rule` (締め日支払い日, closing/pay-date rule) automatically. Remember from Lesson 13 that `Employee`'s `default_scope` hides employees without one!
- `:with_rules` creates all the "rule" records (profile, basic pay, commute, insurance...) that a real employee has. Use it when the code under test reads salary/insurance data.

### 4.2 Using factories in specs

`config.include FactoryBot::Syntax::Methods` (`spec/rails_helper.rb:63`) lets you call these directly:

| Method | What it does | Saved to DB? |
|---|---|---|
| `build(:employee, company: company)` | Makes a new object in memory | ❌ |
| `create(:employee, company: company)` | Builds **and saves** it | ✅ |
| `build_stubbed(:company)` | Fake "saved-looking" object with an id, no DB at all (rare here) | ❌ |
| `create_list(:employee, 2, :with_rules, company: company)` | Creates N records | ✅ |

Real example with `create_list` + trait:
```ruby
# spec/controllers/api/private/yearend_adjustments/bulk_unfix_controller_spec.rb:6-9
let(:company) { default_company }
let!(:cya) { create(:company_yearend_adjustment, company: company, year: year) }
let(:employees) { create_list(:employee, 2, :with_rules, company: company) }
let(:params) { { employee_ids: employees.map(&:id) } }
```

**Prefer `build` when you don't need the DB** (e.g. validation tests: `subject { build(:approver_access_control, ...) }`). It's faster.

---

## 5. Request specs: testing an endpoint end to end

### 5.1 Where and how (`spec/controllers/README.md`)

- **Endpoints** are tested with **request specs**: `type: :request`. They live in **`spec/controllers/`** (not `spec/requests/`; Danger warns if you put them there).
- **Controller specs** (`type: :controller`) are only for **base classes and concerns**, and must be listed in `ALLOW_CONTROLLER_SPEC_LIST` (`spec/rails_helper.rb:245`). This is enforced:

```ruby
# spec/rails_helper.rb:362-366
config.before(:all, type: :controller) do
  if ::ALLOW_CONTROLLER_SPEC_LIST.none?(self.class.metadata[:file_path].to_s)
    raise "#{self.class.metadata[:file_path]} は controller spec で記述しています。..."
  end
end
```
- If a login session is needed, call `log_in_as_an_admin_user` or `log_in_as_a_self_only` in the **top-level `before`**, then override inside contexts when needed.
- `FileRepository.upload_file`, `Webhook::EmployeeNotificationService` and authentication are **already stubbed globally** (`spec/rails_helper.rb:222-236`), so you don't need to stub them yourself.
- **Don't write tests that just copy `config/permission.yaml`** ("role X can call endpoint Y"). That file *is* the specification; repeating it in tests adds nothing.

`log_in_as_an_admin_user` comes from `spec/support/api_driver.rb` (included for request specs by `config.include ApiDriver, type: :request`, `rails_helper.rb:61`):

```ruby
# spec/support/api_driver.rb:36-42
def log_in_as_an_admin_user
  log_in_as('company_admin')
end

def log_in_as_a_self_only
  log_in_as('self_only')
end
```
`log_in_as(authority)` (line 11) stubs the permission service (Sekisyo) and makes `current_auth_user` / `current_auth_company` return fake logged-in objects for `default_company`. *company_admin* = 管理者 (full access); *self_only* = 本人のみ (an employee who can only see their own data).

### 5.2 Line by line: the `EmployeeDisplayNames` request spec

Remember the controller from Lesson 15:
```ruby
# app/controllers/api/private/employee_display_names_controller.rb:7-15
def show
  member =
    ::HrMaster::Infrastructure::Repositories::MemberRepository.find_one_by_employee_id(
      company_id: current_company.id,
      employee_id: show_params[:id].to_i,
    )

  render json: { id: member.employee_id, display_name: member.display_name }
end
```

Its spec, the whole file:
```ruby
# spec/controllers/api/private/employee_display_names_controller_spec.rb:1-40
 1 require 'rails_helper'
 2
 3 describe Api::Private::EmployeeDisplayNamesController, type: :request do
 4   let(:company) { default_company }
 5   before { log_in_as_an_admin_user }
 6
 7   describe '#show' do
 8     subject { get '/api/p/employee_display_names/1', as: :json }
 9
10     before do
11       allow(::HrMaster::Infrastructure::Repositories::MemberRepository).to receive(:find_one_by_employee_id).and_return(
12         HrMaster::Domain::Models::Member.new(employee_id: 1, display_name: '給与太郎', employee_number: 'EMP001'),
13       )
14     end
15
16     it do
17       is_expected.to eq 200
18       expect(JSON.parse(response.body, symbolize_names: true)).to eq({ id: 1, display_name: '給与太郎' })
19       expect(::HrMaster::Infrastructure::Repositories::MemberRepository).to have_received(
20         :find_one_by_employee_id,
21       ).with(company_id: company.id, employee_id: 1)
22     end
23
24     context 'when employee is not found' do
25       before do
26         allow(::HrMaster::Infrastructure::Repositories::MemberRepository).to receive(
27           :find_one_by_employee_id,
28         ).and_raise(::HrErrors::UserError.new('従業員が見つかりませんでした。', status: :not_found))
29       end
30
31       it do
32         is_expected.to eq 404
33         expect(JSON.parse(response.body, symbolize_names: true)).to match(hash_including(message: '従業員が見つかりませんでした。'))
34         expect(::HrMaster::Infrastructure::Repositories::MemberRepository).to have_received(
35           :find_one_by_employee_id,
36         ).with(company_id: company.id, employee_id: 1)
37       end
38     end
39   end
40 end
```

| Line | Meaning |
|---|---|
| 3 | Test the controller as a **request spec** (the real route → controller → JSON path). |
| 4 | Use the seeded company. |
| 5 | Every example runs as a logged-in company admin. |
| 8 | `subject` **sends a real GET request** to `/api/p/employee_display_names/1` in JSON format (`as: :json`). In a request spec, `get`/`post`/... **return the HTTP status code as an integer**, so `subject` = `200`, `404`, etc. |
| 11–13 | **Stub**: when `MemberRepository.find_one_by_employee_id` is called, don't run the real one (which would call the nest-auth microservice); just return this fake `Member` named 給与太郎. |
| 17 | The status is 200. (`is_expected` = `expect(subject)`.) |
| 18 | `response` is the last response. Parse its JSON body with `symbolize_names: true` so keys become symbols (`:id` instead of `"id"`), then compare with the expected hash. |
| 19–21 | **Spy check**: the repository *was called*, with exactly `company_id: company.id, employee_id: 1`. This proves the controller converted `'1'` to the integer `1` and passed the logged-in company. |
| 26–28 | In this context, the stub **raises** a `HrErrors::UserError` with status `:not_found` (`and_raise`). |
| 32 | The controller's error handling (`ErrorHandleable`, Lesson 15) turned that into **404**. |
| 33 | `match(hash_including(message: ...))`: the body has *at least* this key/value (other keys like `code`, `fields` are allowed). |

Notice: all three checks for the same situation live in **one** `it` (team rule + aggregate failures).

### 5.3 Another request spec: params, POST, `parsed_body`

```ruby
# spec/controllers/api/private/shift/shift_requests_controller_spec.rb:3-15, 40-50
RSpec.describe Api::Private::Shift::ShiftRequestsController, type: :request do
  let(:company) { default_company }
  let!(:special_holiday_setting) { create(:special_holiday_setting, company: company, name: '結婚休暇', usage_day: 1.0) }
  let(:grpc_client) { double(Omega::GRPC::Client) }
  before do
    log_in_as_an_admin_user
    allow_any_instance_of(ApplicationController).to receive(:applied_plans) { { is_ai_shift_available: true } }
  end

  describe 'GET /api/p/shift/shift_requests' do
    subject { get '/api/p/shift/shift_requests', params: params }

    let(:params) { { start_date: '2024-01-01', end_date: '2024-01-31', group_id: 1 } }
    ...
    context 'gRPC レスポンスの special_leave_settings が空の場合' do
      it 'レスポンスに special_leave_settings: [] が含まれること' do
        allow(Omega::GRPC).to receive(:client).with(:shift).and_return(grpc_client)
        allow(grpc_client).to receive(:list_shift_requests).and_return(
          Proto::Shift::ListShiftRequestsResponse.new({ shift_requests: [], special_leave_settings: [] }),
        )

        subject
        expect(response).to have_http_status(:ok)
        expect(response.parsed_body).to eq('shift_requests' => [], 'special_leave_settings' => [])
      end
    end
```

New things:
- `params: params` sends query parameters (`?start_date=...`).
- `double(Omega::GRPC::Client)` creates a **plain fake object**. It knows no methods until you stub them.
- `allow(Omega::GRPC).to receive(:client).with(:shift).and_return(grpc_client)`: when code asks for the `:shift` gRPC client (the shift microservice, Lesson 19), give it our fake.
- `allow_any_instance_of(ApplicationController).to receive(:applied_plans)`: stubs that method on **every** controller instance (here: pretend the company's plan includes AI shift).
- `have_http_status(:ok)` checks `response` status (`:ok` = 200). It's an alternative to `is_expected.to eq 200`.
- `response.parsed_body` = the JSON body already parsed (string keys).

POST with a JSON body:
```ruby
# shift_requests_controller_spec.rb:116-124
subject do
  post '/api/p/shift/shift_requests/apply_to_shift',
       params: {
         group_id: 1,
         start_date: '2024-09-01',
         end_date: '2024-09-30',
       },
       as: :json
end
```

---

## 6. Doubles, stubs, mocks: the full toolkit

| Syntax | Kind | Meaning |
|---|---|---|
| `double('file', size: 1.megabyte)` | double | Fake object with only these methods. Unknown methods → error. |
| `instance_double(SomeClass)` | *verified* double | Like `double`, but RSpec **checks** that stubbed methods really exist on `SomeClass`. Safer; typos fail. |
| `allow(obj).to receive(:m).and_return(x)` | stub | Calls to `obj.m` return `x`. |
| `allow(obj).to receive(:m) { x }` | stub | Same, block form. |
| `.and_raise(Error.new(...))` | stub | Calls raise that error. |
| `.and_call_original` | stub | Watch the call but still run the real method. |
| `.with(a: 1)` | constraint | Only matches calls with those arguments. |
| `expect(obj).to receive(:m).with(...)` | **mock** (set *before* the call) | The test **fails** if `m` isn't called like that. |
| `expect(obj).to have_received(:m).with(...)` | **spy** (checked *after*) | Requires an earlier `allow`. |
| `.once`, `.twice` | count | Exact number of calls. |
| `anything` | argument matcher | "Any value is fine here." |

`instance_double` example:
```ruby
# spec/controllers/api/private/e_filing_permission_controller_spec.rb:9-16
it do
  service_double = instance_double(Application::Auth::ValidateEFilingUsableService)
  allow(service_double).to receive(:execute).and_return(true)
  allow(Application::Auth::ValidateEFilingUsableService).to receive(:new).and_return(service_double)

  expect(subject).to eq 200
  expect(JSON(response.body)).to eq({ 'is_usable' => true })
end
```
"When the controller does `ValidateEFilingUsableService.new(...)`, give it the fake; its `execute` returns `true`; so the endpoint must respond `{ "is_usable": true }`."

**Mock style** (expectation set *before* the action):
```ruby
# spec/controllers/api/private/plans/cancellations_controller_spec.rb:19-26
it do
  expect(::Plan::CancelService).to receive(:new).with(
    company: company,
    user_id: anything,
    subscription_number: 'A-S00000001',
  )
  expect(subject).to eq 204
end
```
Order matters: the `expect ... to receive` line comes **before** `subject` runs.

**Spy style** (stub first, check after): the `have_received` lines in §5.2. Both styles are fine; the spy style reads in "arrange → act → assert" order.

### 6.1 Faking HTTP: WebMock `stub_request`

Tests must never call real external servers. WebMock (`require 'webmock/rspec'` in `rails_helper`) blocks them, and `stub_request` fakes the reply:

```ruby
# spec/lib/freee_sign_client/revoke_token_client_spec.rb:9-28
before do
  stub_const('FREEE_SIGN_CLIENT_ID', 'CLIENT_ID')
  stub_const('FREEE_SIGN_CLIENT_SECRET', 'CLIENT_SECRET')
end

context 'when success' do
  before do
    stub_request(:post, "#{FREEE_SIGN_API_URL}/oauth/revoke").with(
      body: {
        'client_id' => 'CLIENT_ID',
        'client_secret' => 'CLIENT_SECRET',
        'token' => 'TOKEN',
      },
      headers: {
        'Content-Type' => 'application/x-www-form-urlencoded',
      },
    ).to_return(body: '{}', status: 200)
  end

  it { expect { subject }.not_to raise_error }
end
```
- `stub_const('NAME', value)` temporarily replaces a constant (here: secret settings) for this test.
- `stub_request(:post, url).with(body:, headers:).to_return(body:, status:)`: "If the code POSTs to this URL with this body and headers, pretend the server answered 200 `{}`."

---

## 7. Data Driven Tests (DDT)

Official rule: `docs/knowledge/general/coding-standards/data-driven-test.md`. Write parameterized tests as a **table of data + a loop**.

| Number of cases | Style |
|---|---|
| **3 or fewer** | **Simple**: an array of arrays `[[input, expected, 'description'], ...]` |
| **4 or more** | **Basic**: an array of hashes `[{ description: ..., input: ..., expected: ... }, ...]` |

Benefits (from the doc): the table makes differences between cases easy to see; fewer `let`s means smaller state scope; adding a case = adding a row.

Real simple-style example:
```ruby
# spec/modules/application/infrastructure/repositories/freee_sign_team_repository_spec.rb:4-27
describe '.find_one_by_access_token_and_company_id' do
  subject { described_class.find_one_by_access_token_and_company_id(access_token: 'dummy_token', company_id:) }

  let(:company_id) { 'DUMMY_COMPANY_ID' }

  # stree-ignore
  test_data = [
    ['client responds with a team hash', { id: 1, name: 'チーム' }, true],
    ['client responds with no data',     nil,                       false],
  ]
  test_data.each do |(title, data, expected)|
    context title do
      before do
        client_double = instance_double(FreeeSignClient)
        allow(client_double).to receive(:get_team) { data }
        allow(FreeeSignClient).to receive(:new) { client_double }
      end

      it do
        is_expected.to eq expected ? ::Application::Domain::Models::FreeeSignTeam.new(id: 1, name: 'チーム') : nil
        expect(FreeeSignClient).to have_received(:new).with(company_id:, access_token: 'dummy_token').once
      end
    end
  end
end
```
- `test_data` is a normal local variable (not `let`), defined at group level.
- `test_data.each do |(title, data, expected)|` **destructures** each row into 3 variables (Lesson 5).
- `# stree-ignore` stops the **Syntax Tree** formatter (`bin/stree`) from reformatting the aligned table. Keep your columns lined up so it reads like a table.
- `company_id:` inside the call is Ruby 3.1 shorthand for `company_id: company_id`.

Basic (hash) style shape, illustrative example from the official doc:
```ruby
test_cases = [
  { description: '年収300万円の場合', annual_income: 3_000_000, dependents: 0, expected_tax: 102_500 },
  { description: '年収500万円、扶養1人の場合', annual_income: 5_000_000, dependents: 1, expected_tax: 142_500 },
  # ...
]

test_cases.each do |test_case|
  context test_case[:description] do
    it '正しい所得税額を計算する' do
      calculator = TaxCalculator.new(annual_income: test_case[:annual_income], dependents: test_case[:dependents])
      expect(calculator.calculate_income_tax).to eq(test_case[:expected_tax])
    end
  end
end
```

---

## 8. Time: `travel_to` and `freeze_time`

Code that depends on "now" (expiry, 年度 year, pay dates) needs a fixed clock. `config.include ActiveSupport::Testing::TimeHelpers` (`rails_helper.rb:60`) gives you:

| Helper | Meaning |
|---|---|
| `travel_to(time)` | From now on in this example, `Time.current` returns `time`. Time is frozen there. |
| `travel_to(time) do ... end` | Only inside the block. |
| `freeze_time` | Stop the clock at the current moment. |

Non-block (used in `before` and inside `it`): `approver_access_control_spec.rb:44, 54, 59` (Lesson 20 §6).

Block form, returning the block's value:
```ruby
# spec/modules/employee_master/infrastructure/repositories/zenshoku_gensenchoshu_hyo_repository_spec.rb:34-42
res =
  travel_to(Time.zone.local(2023, 8, 27, 21, 38, 8)) do
    described_class.upload_file(
      company_id: company.id,
      employee_id: employee.id,
      year: 2022,
      file: Rack::Test::UploadedFile.new('spec/fixtures/images/sample.jpg', 'image/jpeg'),
    )
  end
```
Later the test checks that the saved file name contains `_20230827_213808_000.jpeg`. That's only possible because the time was fixed.

`freeze_time`:
```ruby
# spec/lib/batches/send_scheduled_healthcheck_reminder_mail_spec.rb:7-16
it do
  freeze_time

  allow(Healthcare::Application::Services::SendScheduledHealthcheckReminderMailService).to receive(:execute).with(
    target_datetime: Time.zone.now,
  )
  allow(Omega::DynamoLock).to receive(:with_lock)

  expect { subject }.not_to raise_error
end
```
With the clock frozen, `Time.zone.now` in the test and in the code are *the same instant*, so `.with(target_datetime: Time.zone.now)` can match.

---

## 9. Reuse: `shared_examples`, `it_behaves_like`, `shared_context`

A **shared example** group is defined once and "plugged into" several specs.

```ruby
# spec/modules/application/infrastructure/repositories/shared_examples/multi_sheet.rb:1-15
# MEMO: 利用側で subject を定義すること
shared_examples 'MultiSheetKomokuRepository の引数チェック' do |include_invisible|
  before do
    allow(::Application::Infrastructure::Repositories::MultiSheetKomokuRepository).to receive(
      :list_by_company,
    ).and_call_original
  end

  it '引数が正しいこと' do
    subject
    expect(::Application::Infrastructure::Repositories::MultiSheetKomokuRepository).to(
      have_received(:list_by_company).with(company_id: company.id, include_invisible: include_invisible),
    )
  end
end
```
Used like this:
```ruby
# spec/modules/application/infrastructure/repositories/multi_sheet_shukei_repository_spec.rb:4, 31
require "#{File.dirname(__FILE__)}/shared_examples/multi_sheet.rb"
...
    it_behaves_like 'MultiSheetKomokuRepository の引数チェック', true
```
- The comment says "the caller must define `subject`": shared examples use the caller's `subject`, `let`s (here `company`).
- `true` is passed in as the block parameter `include_invisible`.
- `shared_context 'name' do ... end` + `include_context 'name'` works the same way but is for shared **setup** (`let`/`before`) rather than tests.

When you see `it_behaves_like '...'` in a spec, search the string to find the definition.

---

## 10. Feature flags in tests

From `docs/knowledge/general/coding-standards/feature-flag.md`: **test code must separate the new-behavior and old-behavior `context`s.** Real example:

```ruby
# spec/values/payroll/v2026/shinzoku_spec.rb:55-78 (excerpt)
context 'when open_nencho_2026_enabled フラグがONの場合' do
  before do
    allow(Omega::FeatureFlag).to receive(:active?).with('open_nencho_2026_enabled', namespace: :payroll).and_return(
      true,
    )
  end
  ...
end

context 'when open_nencho_2026_enabled フラグがOFFの場合' do
  before do
    allow(Omega::FeatureFlag).to receive(:active?).with('open_nencho_2026_enabled', namespace: :payroll).and_return(
      false,
    )
  end
  ...
```
When the flag is later removed, you delete one whole context. Clean and safe.

Also know about the CI job **`rspec_with_release_flag_true`** (`.circleci/config.yml:320`): it runs specs with `RSPEC_ALL_RELEASE_FLAG_ENABLED=1`, so all sysparam release flags are **ON** (`spec/rails_helper.rb:157-175` forces boolean flags to `true`). Your test can pass normally and still fail in this job if it only works with the flag OFF. Stub the flag explicitly in each context to make the intent clear.

Also, `spec/rails_helper.rb:199-201` globally does `allow(Omega::Bugcap).to receive(:notify)` and `allow(Omega::FeatureFlag).to receive(:active?).and_call_original`, so error notifications never go out from tests, and feature flags behave normally unless you override them.

---

## 11. Service specs: `type: :service` and `login_user`

`config.extend ServiceMacros, type: :service` (`rails_helper.rb:80`) adds a **macro** `login_user` for service specs:

```ruby
# spec/support/service_macros.rb:1-6 (excerpt)
module ServiceMacros
  def login_user(role = 'company_admin')
    let(:company) { default_company }
    let(:cya) { create(:company_yearend_adjustment, company: company) }
    let(:auth_company) { build_auth_company(company) }
    let(:auth_user) { build_auth_user(auth_company, role) }
```
Calling `login_user` at the top of a `describe ..., type: :service do` defines `company`, `auth_user`, `cps`, and so on. You'll mostly see it in older service specs. Newer ones just use `let(:company) { default_company }` (e.g. `upload_file_service_spec.rb:6`).

---

## 12. Practice (15 min)

**Practice 1.** Write a factory for an (illustrative) `:holiday_request` with: `sequence(:title)` producing `"休暇申請1"`, `"休暇申請2"`...; `status { 'draft' }`; a trait `:approved` that sets `status { 'approved' }` and `approved_at { Time.current }`.

**Practice 2.** Rewrite as a Data Driven Test (simple style, 3 cases):
```ruby
context 'floor' do
  it { expect(Rounder.round(1.5, :floor)).to eq 1 }
end
context 'round' do
  it { expect(Rounder.round(1.5, :round)).to eq 2 }
end
context 'ceil' do
  it { expect(Rounder.round(1.2, :ceil)).to eq 2 }
end
```

**Practice 3.** What's wrong with this spec in freee-payroll? (Find 3 problems.)
```ruby
describe Api::Private::CommentsController, type: :controller do
  let(:company) { create(:company) }
  let(:comment) { create(:comment, id: 1, company: company) }

  it { expect(response.status).to eq 200 }
  it { expect(response.parsed_body).to be_present }
end
```

<details><summary>Practice answers</summary>

1.
```ruby
FactoryBot.define do
  factory :holiday_request do
    sequence(:title) { |n| "休暇申請#{n}" }
    status { 'draft' }

    trait :approved do
      status { 'approved' }
      approved_at { Time.current }
    end
  end
end
```
2.
```ruby
# stree-ignore
test_data = [
  [1.5, :floor, 1, 'floor'],
  [1.5, :round, 2, 'round'],
  [1.2, :ceil,  2, 'ceil'],
]
test_data.each do |(value, round_type, expected, title)|
  context title do
    it { expect(Rounder.round(value, round_type)).to eq expected }
  end
end
```
3. (a) An endpoint must use `type: :request`, not `:controller`. (b) `create(:company)`: use `default_company`. (c) `id: 1` is forbidden in factories. Also: there's no login (`log_in_as_an_admin_user`) and no request is sent (no `subject { get ... }`), and the two `it`s check the same situation, so they should be one example.
</details>

---

## 13. Summary

- **Team rules:** no explicit ids; DDT for parameterized tests (≤3 → arrays, ≥4 → hashes); `aggregate_failures` is global, so combine same-condition checks in one `it`; don't transcribe implementation; no custom matchers; use **`default_company`**.
- **FactoryBot:** `factory`, default blocks, `sequence`, `trait`, `association`, `after(:create)`, `class:`; use via `build` / `create` / `build_stubbed` / `create_list`; `:employee, :with_rules`.
- **Request specs** live in `spec/controllers/` with `type: :request`; log in with `log_in_as_an_admin_user`; `get/post ... as: :json` returns the status; read the body with `JSON.parse(response.body, symbolize_names: true)` or `response.parsed_body`.
- **Doubles:** `double`, `instance_double`; `allow...to receive...and_return/and_raise`; mocks `expect...to receive` (before) vs spies `have_received` (after); `stub_request` for HTTP; `stub_const`.
- **Time:** `travel_to` (block / non-block), `freeze_time`.
- **Reuse:** `shared_examples` + `it_behaves_like`, `shared_context`.
- **Flags:** separate ON/OFF contexts; remember the `rspec_with_release_flag_true` CI job.

➡️ Now take **Test21.md**.
