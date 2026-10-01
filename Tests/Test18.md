# Test 18 — Layered Architecture in `app/modules`

**Rules:** Don't look at Lesson18.md. Then check `Answers18.md`.
**Total:** 42 points

---

## Part A — Multiple choice (1 point each)

**1.** A *domain* is:
- a) A website address
- b) An area of the business with its own rules and words (e.g. Payroll)
- c) A database
- d) A Ruby gem

**2.** An *entity* is equal to another entity when:
- a) All their fields are equal
- b) Their ids (identity) are equal
- c) They were created at the same time
- d) Never

**3.** A *value object* is:
- a) Mutable and identified by id
- b) Immutable and equal by its values
- c) Always an ActiveRecord model
- d) A controller

**4.** Which layer is the ONLY one allowed to touch ActiveRecord?
- a) domain
- b) application
- c) infrastructure
- d) all of them

**5.** Which dependency direction is correct?
- a) domain → infrastructure
- b) application → domain ← infrastructure
- c) domain → application
- d) infrastructure → application → domain → infrastructure

**6.** Where do controllers and serializers live?
- a) `app/modules/<domain>/presentation/controllers`
- b) Outside `app/modules`, in `app/controllers` / `app/serializers`
- c) `app/modules/common`
- d) `lib/`

**7.** A *DAG* is:
- a) A graph of arrows with no cycles
- b) A type of database index
- c) A Sorbet type
- d) A CI job

**8.** According to `.rubocop.yml`, which reference is **banned**?
- a) `Payroll::X` using `Attendance::Y`
- b) `Attendance::X` using `EmployeeMaster::Y`
- c) `Attendance::X` using `Payroll::Y`
- d) `Payroll::X` using `EmployeeMaster::Y`

**9.** A *factory* in `infrastructure/factories` is used to:
- a) Create test data
- b) Build a domain model from raw data such as an ActiveRecord row
- c) Start background jobs
- d) Generate TypeScript code

**10.** A *query service* returns:
- a) Always a domain entity
- b) Read-only data, often plain hashes, for display
- c) HTTP responses
- d) Nothing; it only writes

**11.** When unsure whether to make something an entity or a value object, the team rule is:
- a) Entity
- b) Value object
- c) ActiveRecord model
- d) Ask in Slack every time

---

## Part B — Fill in the blanks (1 point each)

**12.** The same words used by business people and in the code are called the ______ ______.

**13.** The cop that forbids reverse dependencies between domains is `Dependency/______`.

**14.** The allowed domain order is `Payroll → ______ → ______`.

**15.** `app/modules/______` holds logic that fits no domain and must not be referenced by other domains; `app/modules/______` holds cross-domain logic used by all domains.

**16.** In `::Payroll::Domain::Models::CompanyDailyProrateSetting`, the leading `::` means start from the ______ namespace.

---

## Part C — Fill in the missing code (2 points each)

**17.** Fill in the layer names in this path so it's a repository:
`app/modules/payroll/______/______/company_allowance_repository.rb`

**18.** Complete the `T::Enum`:

```ruby
class Frequency < T::______
  ______ do
    ThisMonth = new(:this_month)
    Monthly = ______(:monthly)
  end
end
```

**19.** Complete the controller so a domain validation error becomes a 400 (fill the blank line):

```ruby
  def create
    allowance = ::Payroll::Application::Services::CreateCompanyAllowanceService.execute(company_id: current_company.id, params: permitted_params.to_h, variable_params: permitted_variable_params.to_h)
    render json: build_response(allowance), status: :ok
  ______ ::ActiveModel::ValidationError => e
    raise_invalid_parameter!('手当の設定内容が不正です。', e.message)
  end
```

---

## Part D — Write code (5 points each)

**20.** Write a **domain-layer value object** `Attendance::Domain::Models::WorkPeriod` in nested-module style with:
- `const :start_time, String` and `const :end_time, String`
- a validation that both are present
- include the freee-payroll value-object mixin

**21.** Write a **repository** `Attendance::Infrastructure::Repositories::WorkPeriodRepository` with one method `find_one_by_company_id(company_id:)` that:
- loads `Company.find(company_id).work_period_setting` (an AR record, may be `nil`)
- returns a `WorkPeriod` value object. If the record is `nil`, default to `'09:00'`–`'18:00'`

(Use `class << self`. Sorbet `sig` optional.)

---

## Part E — Answer in a sentence or two (2 points each)

**22.** Why do the arrows point *into* the domain layer (why should the domain depend on nothing)?

**23.** `UpdateCompanyDailyProrateSettingService` has a `@deprecated` comment. Why, and what's the preferred argument style?

**24.** Why does `CompanyAllowanceFactory` define its own `InvalidCombinationError` instead of raising a generic `StandardError`?

---

## Part F — Read real code (4 points)

**25.** From `app/modules/payroll/application/services/update_company_daily_prorate_setting_service.rb`:

```ruby
            company_daily_prorate_setting =
              ::Payroll::Domain::Models::CompanyDailyProrateSetting.new(company_id: company.id, round_type: round_type)
            if current_company_daily_prorate_setting != company_daily_prorate_setting
              ActiveRecord::Base.transaction do
                ::Payroll::Infrastructure::Repositories::CompanyDailyProrateSettingRepository.save(
                  company_daily_prorate_setting: company_daily_prorate_setting,
                )

                cps_ids = company.company_payroll_statements.variable.pluck(:id)
                ::Application::CalcPayrollStatementsService.new(:none, company.id, cps_id: cps_ids).execute
              end
            end
```

a) Which layer is this file in?
b) What does `!=` compare here, and why does that work (hint: what kind of object is it)?
c) What happens if `round_type` is `'banana'`?
d) Why are `save` and the recalculation inside the same transaction?
