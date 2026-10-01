# Test 21 — RSpec in freee-payroll

**Rules:** Don't look at Lesson21.md while answering. Write your answers under each question. Then check `Answers21.md`.
**Total:** 42 points

---

## Part A — Multiple choice (1 point each)

**1.** What is a *flaky test*?
- a) A test with no expectations
- b) A test that sometimes passes and sometimes fails without any code change
- c) A test that takes more than 10 seconds
- d) A test that uses a factory

**2.** In freee-payroll, which company should a normal spec use?
- a) `create(:company)`
- b) `build(:company)`
- c) `default_company`
- d) `Company.new`

**3.** Where do request specs for Private API endpoints live?
- a) `spec/requests/api/private/`
- b) `spec/controllers/api/private/` with `type: :request`
- c) `spec/api/`
- d) `test/controllers/`

**4.** What does `instance_double(SomeService)` add compared to `double('x')`?
- a) Nothing; they're identical
- b) It saves the double to the database
- c) It verifies that the methods you stub really exist on `SomeService`
- d) It automatically calls the real methods

**5.** What does `and_raise(...)` do in `allow(repo).to receive(:find_one_by_employee_id).and_raise(error)`?
- a) Checks that the method raised an error
- b) Makes the stubbed method raise that error when called
- c) Retries the call if it errors
- d) Logs the error

**6.** What is the difference between `expect(x).to receive(:m)` and `expect(x).to have_received(:m)`?
- a) No difference
- b) `receive` is set *before* the action (mock); `have_received` is checked *after*, and needs an earlier `allow` (spy)
- c) `have_received` only works on doubles
- d) `receive` is only for class methods

**7.** Which tool fakes an external HTTP response in a test?
- a) `stub_const`
- b) `travel_to`
- c) `stub_request` (WebMock)
- d) `create_list`

**8.** Why can you write several `expect` lines in one `it` and still see *all* failures?
- a) Because of `use_transactional_fixtures`
- b) Because `aggregate_failures` is enabled globally in `rails_helper.rb`
- c) Because of `# stree-ignore`
- d) You can't; RSpec always stops at the first failure

**9.** In `spec/factories.rb`, when is giving an explicit `id` to `create` allowed?
- a) Never
- b) Always
- c) When the spec file path contains `import` or `mass_update`
- d) Only in request specs

**10.** Data Driven Test style: you have **5** test cases. Which style should you use?
- a) Simple (array of arrays)
- b) Basic (array of hashes with a `description:` key)
- c) Separate `context` blocks with lots of `let`
- d) A custom matcher

---

## Part B — Fill in the blanks (1 point each blank)

**11.** `create(:employee, company: company)` saves to the DB; `______(:employee, company: company)` only builds it in memory; `______(:employee, 3, company: company)` creates three.

**12.** In a factory, `______(:name) { |n| "test#{n}" }` produces unique values, and `______ :approved do ... end` defines a named variation.

**13.** In a request spec, `get '/api/p/...'` returns the HTTP ______ as an integer, so `is_expected.to eq 200` works.

**14.** To make keys symbols when parsing a response: `JSON.parse(response.body, ______: true)`.

**15.** The comment `# ______` above an aligned DDT table stops the Syntax Tree formatter from reformatting it.

**16.** The CI job `rspec_with_release_flag_______` runs specs with all release flags ON.

---

## Part C — Fill in the missing code (2 points each)

**17.** Complete the factory:
```ruby
FactoryBot.define do
  factory :hr_portal_action do
    ______(:name) { |n| "アクション#{n}" }
    description { 'アクション説明' }
    action_type { 'attrition_risk' }

    ______ :with_sub_actions do
      ______(:create) { |action| create_list(:hr_portal_sub_action, 2, hr_portal_action: action) }
    end
  end
end
```

**18.** Complete this request spec so it logs in as an admin, stubs the repository to return a fake member, and checks status + body:
```ruby
describe Api::Private::EmployeeDisplayNamesController, type: :request do
  let(:company) { ______ }
  before { ______ }

  describe '#show' do
    subject { ___ '/api/p/employee_display_names/1', as: :json }

    before do
      ______(::HrMaster::Infrastructure::Repositories::MemberRepository).to ______(:find_one_by_employee_id).______(
        HrMaster::Domain::Models::Member.new(employee_id: 1, display_name: '給与太郎', employee_number: 'EMP001'),
      )
    end

    it do
      is_expected.to eq ___
      expect(JSON.parse(response.body, symbolize_names: true)).to eq({ id: 1, display_name: '給与太郎' })
    end
  end
end
```

**19.** Complete the time-freezing code so the saved file name will always contain `20230827`:
```ruby
res =
  ______(Time.zone.local(2023, 8, 27, 21, 38, 8)) ___
    described_class.upload_file(company_id: company.id, employee_id: employee.id, year: 2022, file: file)
  ___
```

---

## Part D — Write the code (4 points each)

**20.** Write a Data Driven Test (simple style) for this illustrative method. Cover 3 cases: `'floor'` → 1, `'round'` → 2, `'ceil'` → 2 for input `1.5`.
```ruby
class DailyProrate
  def self.apply(value:, round_type:)
    case round_type
    when 'floor' then value.floor
    when 'round' then value.round
    when 'ceil' then value.ceil
    end
  end
end
```

**21.** `Api::Private::CommentsController#destroy` does this (real code, `app/controllers/api/private/comments_controller.rb:45-59`):
```ruby
def destroy
  comment =
    Application::Infrastructure::Repositories::CommentRepository.find_one_by_id(
      company_id: current_company.id,
      id: params[:id].to_i,
    )

  raise HrErrors::UserError.new('コメントの削除権限がありません。', status: :forbidden) if comment.user_id != current_auth_user.id

  ActiveRecord::Base.transaction do
    Application::Infrastructure::Repositories::CommentRepository.delete(comment: comment)
  end

  head :no_content
end
```
Write a request spec with a top-level login and **two** contexts:
- the author deletes their own comment → `204` and the company has 0 comments
- another user's comment (`user_id: current_auth_user.id + 1`) → `403` and the comment still exists

Use `default_company`, `create(:comment, ...)`, and `delete "/api/p/comments/#{...}"`. (`current_auth_user` is available in request specs.)

---

## Part E — Answer in a sentence or two (2 points each)

**22.** Why does the team use `default_company` instead of `create(:company)` in most specs? When is `create(:company)` acceptable?

**23.** Why shouldn't you write tests that just copy `config/permission.yaml`, or that transcribe the implementation line by line?

**24.** Why should tests for code behind a feature flag have separate ON and OFF contexts?

---

## Part F — Read real code (4 points)

**25.** Real code from `spec/controllers/api/private/comments_controller_spec.rb` (lines 1–9 and 187–207, lightly trimmed). Explain: (a) what kind of spec this is and who is logged in, (b) what `subject` does, (c) what the `before` creates and why `user_id: current_auth_user.id` matters, (d) what the example asserts.

```ruby
describe Api::Private::CommentsController, type: :request do
  let(:company) { default_company }
  before { log_in_as_an_admin_user }

  describe 'destroy' do
    subject { delete "/api/p/comments/#{company.comments.first.id}" }

    before do
      create(
        :comment,
        company: company,
        url: 'payroll_statements#/1/2023/12',
        user_id: current_auth_user.id,
        page_name: '給与明細一覧',
        comment: 'test1',
      )
    end

    context 'コメント作成者が削除できる' do
      it do
        is_expected.to eq 204
        expect(company.comments.count).to eq 0
      end
    end
  end
end
```
