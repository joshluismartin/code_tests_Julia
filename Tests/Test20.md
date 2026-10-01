# Test 20 — RSpec Basics

**Rules:** Don't look at Lesson20.md while answering. Write your answers under each question. Then check `Answers20.md`.
**Total:** 40 points

---

## Part A — Multiple choice (1 point each)

**1.** What is a *regression*?
- a) A test that runs slowly
- b) Something that used to work breaks because of a new change
- c) A kind of database query
- d) A failing test that is ignored

**2.** Which file does almost every spec in freee-payroll `require` on line 1?
- a) `spec_helper`
- b) `rails_helper`
- c) `factory_bot`
- d) `application_record`

**3.** Where should the spec for `app/services/payroll/salaries/unfix_service.rb` live?
- a) `spec/payroll/salaries/unfix_service_spec.rb`
- b) `test/services/payroll/salaries/unfix_service_test.rb`
- c) `spec/services/payroll/salaries/unfix_service_spec.rb`
- d) `app/services/payroll/salaries/unfix_service_spec.rb`

**4.** What is the difference between `let` and `let!`?
- a) `let!` raises an error if the value is nil
- b) `let` runs only when first called; `let!` runs before every example no matter what
- c) `let!` is shared between all examples; `let` is not
- d) There is no difference

**5.** `describe '#execute'` vs `describe '.execute'`: what does the `.` mean?
- a) A private method
- b) A class method
- c) An instance method
- d) A deprecated method

**6.** Which matcher checks that an array has exactly these elements *in any order*?
- a) `eq`
- b) `include`
- c) `contain_exactly`
- d) `match`

**7.** Inside `describe Application::PayrollVerifications::UploadFileService do`, what does `described_class` return?
- a) The string `'UploadFileService'`
- b) The class `Application::PayrollVerifications::UploadFileService`
- c) The current test's name
- d) `nil`

**8.** `is_expected.to eq 200` is shorthand for:
- a) `expect(200).to eq subject`
- b) `expect(response).to eq 200`
- c) `expect(subject).to eq 200`
- d) `expect { subject }.to eq 200`

**9.** After each example finishes, what happens to records it created in the test database?
- a) They stay forever
- b) They are rolled back (removed) so the next test starts clean
- c) They are copied to production
- d) They are soft-deleted with `deleted_at`

**10.** Which CI bot warns on a pull request when a changed `app/` file has no matching spec?
- a) RuboCop
- b) Sorbet
- c) Danger
- d) Committee

---

## Part B — Fill in the blanks (1 point each blank)

**11.** The three nesting words of RSpec are `______` (what is tested), `______` (in which situation), and `______` (one example).

**12.** To check that code raises an error, you must pass a ______ to `expect` (use `{ }`), not parentheses.

**13.** `let` values are ______ (computed only when first used) and ______ (remembered for the rest of that example).

**14.** In Japanese spec names, 正常系 means the ______ path, and 〜の場合 means "in the ______ of ~".

---

## Part C — Fill in the missing code (2 points each)

**15.** Complete the expectation so it says "running subject creates exactly one `Comment`":
```ruby
it do
  expect { subject }.to ______(Comment, :count).______(1)
end
```

**16.** Complete it so it says "running subject raises `HrErrors::UserError` with message 'コメントできませんでした。'":
```ruby
it do
  expect ___ subject ___.to ______(HrErrors::UserError, 'コメントできませんでした。')
end
```

**17.** Fill in the blanks so the inner context *overrides* `speaker` with `nil` and expects the record to be invalid:
```ruby
subject { approval_request.comments.build({ body: 'hi', speaker: speaker }) }

context 'nilはNG' do
  ______(:speaker) { ______ }
  it { expect(subject).______ be_valid }
end
```

---

## Part D — Write the code (3 points each)

**18.** Here is a small value object (illustrative):

```ruby
class YearMonth
  def initialize(year:, month:)
    raise ArgumentError, 'month must be 1..12' unless (1..12).cover?(month)
    @year = year
    @month = month
  end

  def next
    @month == 12 ? YearMonth.new(year: @year + 1, month: 1) : YearMonth.new(year: @year, month: @month + 1)
  end

  attr_reader :year, :month
end
```

Write a spec file for it with:
- `require 'rails_helper'` and a top-level `describe` using the class
- a `describe '#next'` group with a `subject` that calls `.next` on a `YearMonth` built from `let(:year_month)`
- a context for "normal month" (2026/4 → 2026/5) using `have_attributes`
- a context for December (2026/12 → 2027/1)
- a `describe '.new'` group that checks month 13 raises `ArgumentError`

**19.** Write one `it` block (with `before` if needed) that checks: after running `subject`, `company.payroll_comparison_imports.count` goes up by 1 **and** the last import's `status` is `'tmp_uploaded'`. Use the block form of `change`.

---

## Part E — Answer in a sentence or two (2 points each)

**20.** Why must `expect(subject).to raise_error(SomeError)` be rewritten? What happens if you run it as is?

**21.** When should you use `let!` instead of `let`? Give a situation from a request spec.

**22.** Why are tests especially important in a payroll product like freee人事労務?

---

## Part F — Read real code (4 points)

**23.** This is real code from `spec/models/approval_request/approver_access_control_spec.rb:40-62`. Explain in plain words: (a) what method is tested, (b) what the `before` does, (c) what each of the three examples checks and why the results differ. (`EXPIRE_MINUTES` is 60.)

```ruby
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
end
```
