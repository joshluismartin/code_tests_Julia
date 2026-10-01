# Lesson 18 — Layered Architecture in `app/modules`

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Understand the **DDD (Domain-Driven Design)** vocabulary the team uses, the three layers inside `app/modules/<domain>/`, which way dependencies are allowed to point, and trace one real feature through every layer.
**You need:** Lesson 17 (services, repositories, value objects). Open `~/Josh_Files/freee-payroll/app/modules/`.

---

## 1. The vocabulary (read slowly, it's all used in code and reviews)

| Term | Meaning in plain words | freee-payroll example |
|---|---|---|
| **Domain** | An area of the business, with its own rules and words. | Payroll (給与), Attendance (勤怠), EmployeeMaster (従業員マスタ) |
| **Business domain module** | A folder in `app/modules/` holding one domain's code. | `app/modules/payroll/` |
| **DDD (Domain-Driven Design)** | A way of designing software where the code's structure and names follow the business domain. | This whole lesson |
| **Ubiquitous language** | The *same words* used by business people and in code. | `geppen` (月変) instead of an English phrase nobody at work uses |
| **Domain model** | A class that represents a business concept and its rules, **without** database code. | `Payroll::Domain::Models::CompanyAllowance` (手当マスタ) |
| **Entity** | A domain model with an **identity** (an id). Two entities are "the same" if their ids match, even if other fields differ. Usually changeable. | An allowance #12, even after its name changes |
| **Value object** | A domain model with **no identity**, equal by value and immutable. | `Allowance::PayrollSetting` (a set of true/false flags) |
| **Use case** | One thing a user wants to do with the system. | "Update an allowance" |
| **Application service** | A class that runs one **use case** step by step (loads, calls domain logic, saves). | `UpdateCompanyAllowanceService` |
| **Domain service** | Pure business calculation that doesn't belong to one model. | `CheckCollectionTargetOfChildSupportFundService` |
| **Repository** | Loads/saves domain models (hides the DB). | `CompanyAllowanceRepository` |
| **Factory** | Builds a domain model from raw data (e.g. from an ActiveRecord row). | `CompanyAllowanceFactory` |
| **Query service** | Read-only query that returns plain data for a screen (no domain model needed). | `FetchEmployeePayrollStatementTemplatesQueryService` |
| **Layer** | A group of classes with one kind of job; layers depend on each other in one direction. | application / domain / infrastructure |
| **Dependency** | "A depends on B" = A's code mentions/calls B. If B changes, A might break. | A service calls a repository |
| **Dependency direction** | Which way those arrows are allowed to point. | application → domain ← infrastructure |
| **Namespace** | The `Module::` prefix of a class name. Here it also marks *which domain* a class belongs to. | `Payroll::...`, `Attendance::...` |
| **Boundary** | The edge between two domains/layers that code shouldn't cross the wrong way. | Attendance must not use `Payroll::` |
| **DAG** | *Directed Acyclic Graph*: arrows between things with **no cycles** (you can never follow arrows back to where you started). | Payroll → Attendance → EmployeeMaster |
| **Circular dependency** | A → B and B → A. Bad: you can't change either safely. | Forbidden |

### Entity vs value object: the key idea

```
Entity (identity matters)                 Value object (only values matter)
┌───────────────────────────┐             ┌────────────────────────────┐
│ CompanyAllowance id: 12   │             │ PayrollSetting             │
│ name: "通勤手当"          │  rename →   │ is_affect_income_tax: true │
│ name: "交通費"            │  still #12  │ ...                        │
└───────────────────────────┘             └────────────────────────────┘
  same thing if same id                     same thing if same values
  can change over time                      frozen; "change" = make new one
```

`docs/knowledge/general/architecture/value-object-entity.md` says: **if you're unsure, make it a value object first** (easier equality, clear immutability, easier to change later).

> ⚠️ That doc shows an `ActsAsEntity` module in its examples, but **it does not exist in the code** (only `lib/acts_as_value_object.rb` exists). Real entities are plain classes with `ActiveModel::Validations` and a `validate!` call (you'll see one in §4.3).

---

## 2. The layers

From `docs/knowledge/general/architecture/overview-backend.md`:

```
app/modules/<business_domain>/
├── application/services           # use cases
├── domain/models                  # domain models (Entity / Value Object)
├── domain/services                # domain services
├── infrastructure/repositories    # persistence & talking to external services
├── infrastructure/query_services  # read-only queries
└── infrastructure/factories       # build domain models from ActiveRecord etc.
```

Rules (translated from the doc):

1. **Dependencies go one way only: application → domain ← infrastructure.**
2. **The domain layer depends on nothing else**: no other layers, and no specific technology like ActiveRecord.
3. **Only the infrastructure layer touches ActiveRecord.**
4. The **HTTP layer (controllers, serializers) is not owned by a domain**, so it is *not* in `app/modules`. It stays in `app/controllers` / `app/serializers`.

```
                ┌───────────────────────────────┐
  HTTP layer    │ app/controllers (outside)     │
                └──────────────┬────────────────┘
                               │ calls
                ┌──────────────▼────────────────┐
  application   │ application/services (use case)│
                └───────┬───────────────┬────────┘
                        │ uses          │ uses
          ┌─────────────▼──────┐   ┌────▼──────────────────────┐
  domain  │ domain/models      │◀──│ infrastructure/            │  infrastructure
          │ domain/services    │   │  repositories, factories,  │
          │ (pure Ruby rules)  │   │  query_services            │
          └────────────────────┘   └────────────┬───────────────┘
              ▲ depends on nothing              │ ActiveRecord / gRPC
                                         ┌──────▼──────┐
                                         │ MySQL, other │
                                         │ services     │
                                         └─────────────┘
```

Why point the arrows *into* domain? Because the **business rules are the most valuable, most stable part**. The DB, gRPC, and HTTP change more often. If the domain depends on nothing, a DB change can't break a business rule.

### 2.1 Real folders vary: don't panic

Look at `ls app/modules/*/`. You'll find extras:
- `presentation/` (e.g. `app/modules/payroll/presentation/requests/`) holds request objects.
- Sub-domains like `app/modules/payroll/juminzei/` (住民税, resident tax) have their own `domain/infrastructure/usecase`.
- Year-versioned folders like `app/modules/nenmatsu_chosei/v2026/` exist because year-end-adjustment law changes every year.
- `app/modules/hr_master/` has only `domain/` and `infrastructure/`.

The **core idea (three layers, one direction)** is what matters.

### 2.2 Not-really-domains: `application` and `common`

- `app/modules/application/README.md`: logic that belongs to **no** business domain (or is hard to classify). **No other domain should reference it.**
- `app/modules/common/README.md`: logic used **across all** domains that's hard to classify, e.g. しめしは (締め日支払い日, closing/pay-date settings). "Before adding models here, redo your domain analysis."

---

## 3. Boundaries enforced by RuboCop

Each file in `app/modules/payroll/...` gets the namespace `Payroll::...` (Rails autoloading, Lesson 9). The team uses that to enforce boundaries. `.rubocop.yml:98-116`:

```yaml
Dependency/OverBoundary:
  Enabled: true
  Exclude:
    - 'spec/**/*'
  Rules:
    # Business domain rules
    # The desired dependency direction should be as follows, and an error will occur if there is a reverse dependency:
    # Payroll -> Attendance -> EmployeeMaster
    - BannedConstPatterns: \AAttendance
      FromNamespacePatterns:
        - \AEmployeeMaster(\W|\z)
    - BannedConstPatterns: \APayroll
      FromNamespacePatterns:
        - \AAttendance(\W|\z)
        - \AEmployeeMaster(\W|\z)
    # Layered architecture rules
    - BannedConstPatterns: Repository\z
      FromNamespacePatterns:
        - Serializer\z
```

How to read one rule: *"Constants matching `BannedConstPatterns` must not be used from classes whose namespace matches `FromNamespacePatterns`."*

| Code in... | Uses... | Allowed? |
|---|---|---|
| `Payroll::...` | `Attendance::...` | ✅ (Payroll → Attendance) |
| `Payroll::...` | `EmployeeMaster::...` | ✅ |
| `Attendance::...` | `EmployeeMaster::...` | ✅ |
| `Attendance::...` | `Payroll::...` | ❌ reverse |
| `EmployeeMaster::...` | `Attendance::...` or `Payroll::...` | ❌ reverse |
| `FooSerializer` | `BarRepository` | ❌ serializers must not call repositories |

The business reason, from the doc: payroll needs employee info ("print the employee's name on the payslip"), but the employee master never needs to know about payroll. The arrows form a **DAG**: no cycles.

```
 Payroll ──▶ Attendance ──▶ EmployeeMaster
    └──────────────────────────▲
```

(`\A` in a regex = "start of string", `\z` = "end of string", `\W` = "a non-word character" such as `:`.)

---

## 4. Full slice #1 (simple): the daily-prorate setting (日割り計算設定)

**Feature:** a company picks how to round daily-prorated pay: `floor` (切り捨て), `round` (四捨五入), or `ceil` (切り上げ). Endpoint: `PUT /api/p/company_daily_prorate_setting`.

### 4.1 Controller (HTTP layer, outside modules)

`app/controllers/api/private/company_daily_prorate_settings_controller.rb:12-25`:

```ruby
  def update
    permitted_params = params.permit(:round_type)

    # 設定変更時に全従業員の給与再計算が実行されるが、管理対象外の従業員も含めて処理する必要があるため、部門制御を無効化する
    company_daily_prorate_setting =
      Authz.disable_access_control do
        ::Payroll::Application::Services::UpdateCompanyDailyProrateSettingService.execute(
          company: current_company,
          round_type: permitted_params[:round_type],
        )
      end

    render json: build_response(company_daily_prorate_setting: company_daily_prorate_setting), status: :ok
  end
```

(Comment: "Changing the setting recalculates payroll for **all** employees, including ones outside this user's department scope, so department access control is disabled." You learned `Authz` in Lesson 13.)

### 4.2 Application service (use case)

`app/modules/payroll/application/services/update_company_daily_prorate_setting_service.rb:13-34`:

```ruby
          def execute(company:, round_type:)
            current_company_daily_prorate_setting =
              ::Payroll::Infrastructure::Repositories::CompanyDailyProrateSettingRepository.find_one_by_company_id(
                company_id: company.id,
              )

            company_daily_prorate_setting =
              ::Payroll::Domain::Models::CompanyDailyProrateSetting.new(company_id: company.id, round_type: round_type)
            if current_company_daily_prorate_setting != company_daily_prorate_setting
              ActiveRecord::Base.transaction do
                ::Payroll::Infrastructure::Repositories::CompanyDailyProrateSettingRepository.save(
                  company_daily_prorate_setting: company_daily_prorate_setting,
                )

                # 賃金計算期間のステータスが未確定の給与明細に対して給与計算を実行する
                cps_ids = company.company_payroll_statements.variable.pluck(:id)
                ::Application::CalcPayrollStatementsService.new(:none, company.id, cps_id: cps_ids).execute
              end
            end

            company_daily_prorate_setting
          end
```

Steps: **load** current (repository) → **build** new value object (domain) → **compare** with `!=` (value equality!) → if changed, **save + recalc** inside a **transaction** → return.

Lines 7-9 of the file carry a `@deprecated` note: it takes an ActiveRecord `company` argument; the preferred style is `company_id:` so the application layer doesn't depend on ActiveRecord objects.

### 4.3 Domain model (value object)

`app/modules/payroll/domain/models/company_daily_prorate_setting.rb:15-22`:

```ruby
      class CompanyDailyProrateSetting < T::Struct
        include ActsAsValueObject

        const :company_id, T.untyped
        const :round_type, T.untyped

        validates :round_type, inclusion: %w[floor round ceil] # 日割り計算の端数処理方法. floor: 切り捨て, round: 四捨五入, ceil: 切り上げ
      end
```

No ActiveRecord here. `CompanyDailyProrateSetting.new(company_id: 1, round_type: 'banana')` raises right away, so **invalid states can't exist**. (Its file comment explains it's conceptually an Entity, implemented as a value object because there's no entity base class and immutability keeps invariants safe.)

### 4.4 Infrastructure: repository

`app/modules/payroll/infrastructure/repositories/company_daily_prorate_setting_repository.rb:8-29`:

```ruby
          def find_one_by_company_id(company_id:)
            company_daily_prorate_setting = Company.find(company_id).company_daily_prorate_setting
            if company_daily_prorate_setting.nil?
              ::Payroll::Domain::Models::CompanyDailyProrateSetting.new(company_id: company_id, round_type: 'floor')
            else
              ::Payroll::Domain::Models::CompanyDailyProrateSetting.new(
                company_id: company_daily_prorate_setting.company_id,
                round_type: company_daily_prorate_setting.round_type,
              )
            end
          end

          # @params [Domain::Models::CompanyDailyProrateSetting] 事業所日割り計算設定
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

This is the **only** place in the slice that touches ActiveRecord (`Company.find`, `update!`). It translates **AR row ⇄ domain model**. Note the business default: no row → `'floor'`.

### 4.5 The ActiveRecord model (plain Rails, outside modules)

`app/models/company_daily_prorate_setting.rb` has `belongs_to :company`, an `enum :round_type`, and `validates :round_type, presence: true`.

> Two classes with the same short name: `::CompanyDailyProrateSetting` (AR model, the table) and `::Payroll::Domain::Models::CompanyDailyProrateSetting` (domain model). The **leading `::`** means "start from the top-level namespace". Always read the full name!

---

## 5. Full slice #2 (bigger): allowances (手当マスタ)

Allowances (手当) are extra pay items like commuting allowance (通勤手当). One allowance has a name, a frequency, calculation settings, and flags for which taxes/insurances it affects.

### 5.1 Controller: create + turning domain errors into 400

`app/controllers/api/private/company_allowances_controller.rb:61-74`:

```ruby
  def create
    company_allowance =
      ::Payroll::Application::Services::CreateCompanyAllowanceService.execute(
        company_id: current_company.id,
        params: permitted_params.to_h,
        variable_params: permitted_variable_params.to_h,
      )

    render json: build_response(company_allowance), status: :ok
  rescue ::Payroll::Infrastructure::Factories::CompanyAllowanceFactory::InvalidCombinationError,
         ::ActiveModel::ValidationError => e
    # ドメインモデルの初期化が持つ検証（calc_type ごとの許容値・範囲）を API 境界で 400 に差し替える。
    raise_invalid_parameter!('手当の設定内容が不正です。再度読み込みをしても解消しない場合は、サポートセンターまでご連絡ください。', e.message)
  end
```

The comment: *"At the API boundary, replace the domain model's initialization validation errors with a 400."* The domain says "invalid!" by raising; the HTTP layer decides it means **400 Bad Request**.

### 5.2 Application service: update use case

`app/modules/payroll/application/services/update_company_allowance_service.rb` (81 lines). Beginning, lines 13-20:

```ruby
    def execute(company_id:, params:, variable_params:)
      company = Company.find(company_id)

      company_allowance_entity =
        Payroll::Infrastructure::Repositories::CompanyAllowanceRepository.find_one_by_id(
          company_id: company_id,
          company_allowance_id: params[:id],
        )
```

End, lines 60-78:

```ruby
      company_allowance_entity.update_attributes(update_params)

      ActiveRecord::Base.transaction do
        company_allowance =
          Payroll::Infrastructure::Repositories::CompanyAllowanceRepository.update(
            company_id: company_id,
            company_allowance_entity: company_allowance_entity,
          )

        Payroll::CompanyAllowance::RecalcService.new(
          ...
        ).execute

        company_allowance
      end
```

The pattern is **load entity → change entity (it validates itself) → save in a transaction → side effects (recalculate)**.

### 5.3 Domain: the entity and a value object inside it

Entity: `app/modules/payroll/domain/models/company_allowance.rb` (nested-module style, lines 2-14):

```ruby
module Payroll
  module Domain
    module Models
      # 手当マスタ
      class CompanyAllowance
        extend T::Sig
        include ActiveModel::Validations

        validates :name, presence: true, length: { maximum: 240 }
        validates :description, length: { maximum: 255 }
        validate :validate_payroll_setting

        attr_reader :id, :company_id, :name, :description, :frequency, :is_active
```

Its `Frequency` is a **`T::Enum`**, a fixed set of allowed values (lines 47-54):

```ruby
        class Frequency < T::Enum
          enums do
            ThisMonth = new(:this_month)
            Monthly = new(:monthly)
            # 賞与
            Cps = new(:cps)
          end
        end
```

It **changes** through `update_attributes` (lines 120-127), which re-validates:

```ruby
        def update_attributes(params)
          @name = params[:name]
          @description = params[:description]
          @calculation_setting = params[:calculation_setting]
          @payroll_setting = params[:payroll_setting]

          validate!
        end
```

A business rule lives here (lines 131-135): bonus-time allowances (`:cps`) can't count toward the overtime base:

```ruby
        def validate_payroll_setting
          if frequency.serialize == :cps &&
               (payroll_setting.is_affect_overtime_wage_base || payroll_setting.is_affect_attendance_deduction_base)
            errors.add(:payroll_setting, '計算方法の設定値が不正です')
          end
```

Value object inside it: `app/modules/payroll/domain/models/allowance/payroll_setting.rb:2-13`:

```ruby
class Payroll::Domain::Models::Allowance::PayrollSetting < T::Struct
  include ActsAsValueObject
  extend T::Sig

  const :is_affect_income_tax, T::Boolean
  const :is_affect_employment_insurance, T::Boolean
  const :is_affect_labor_insurance, T::Boolean
  const :is_affect_fixed_wage, T::Boolean
  # 以下は支給タイミングが賞与の時は false
  const :is_affect_overtime_wage_base, T::Boolean
  const :is_affect_attendance_deduction_base, T::Boolean
end
```

Two **namespace styles** both appear: nested `module Payroll; module Domain; module Models; class X` and compact `class Payroll::Domain::Models::Allowance::PayrollSetting`. They mean the same class name.

### 5.4 Domain service: pure calculation

`app/modules/payroll/domain/services/check_collection_target_of_child_support_fund_service.rb:14-33` decides whether to collect the new 子ども・子育て支援金 (child support levy):

```ruby
          def execute(birthdate:, is_insured:, hassei_date:)
            return { is_collection_target: false, causes: [], reason_type: :not_entry } unless is_insured

            # 資格喪失日
            disqualification_date =
              Birthdate.new(birthdate: birthdate).date_to_reach(
                age: 75,
                based_on_nenrei_keisan_ni_kansuru_horitsu: false,
              )

            if hassei_date < disqualification_date
              { is_collection_target: true, causes: [] }
            else
              ...
```

No database. Just inputs → answer. That's what "domain layer depends on nothing" looks like. (It uses the `Birthdate` value object from Lesson 17.)

### 5.5 Infrastructure: repository, factory, query service

**Repository** (`company_allowance_repository.rb:9`): `find_one_by_id(company_id:, company_allowance_id:)` reads the AR row, then hands raw fields to the factory.

**Factory**: `app/modules/payroll/infrastructure/factories/company_allowance_factory.rb:2-6`:

```ruby
class Payroll::Infrastructure::Factories::CompanyAllowanceFactory
  # calc_type と amount_source の組み合わせが成立しない場合に投げる。
  # API 境界で rescue して 400 に差し替えるため、narrow に rescue できるクラスとして定義する。
  class InvalidCombinationError < StandardError
  end
```

"Raised when calc_type and amount_source can't go together. Defined as its **own narrow class** so the API boundary can rescue exactly this and turn it into 400." (That's the `rescue` you saw in §5.1.) It builds the entity at line 78: `Payroll::Domain::Models::CompanyAllowance.new(...)`.

**Query service**: `app/modules/payroll/infrastructure/query_services/fetch_employee_payroll_statement_templates_query_service.rb:10-29`:

```ruby
          def execute(company_id:, employee_ids:)
            return [] if employee_ids.empty?

            company = Company.find(company_id)

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
          rescue => e
            raise ::HrErrors::SystemError.new(original_exception: e)
          end
```

Read-only, returns **plain hashes** (not domain models): fast and simple for displaying lists. Any failure becomes a `SystemError` (500).

### 5.6 One request through every layer

```
PATCH /api/p/company_allowances/12   { name: "交通費", ... }
  │
  ▼ Api::Private::CompanyAllowancesController#update     (HTTP; outside modules)
  │   permits params, calls ↓
  ▼ Payroll::Application::Services::UpdateCompanyAllowanceService.execute  (application)
  │   1. Repository.find_one_by_id ────────────────▶ infrastructure
  │          AR row ─▶ CompanyAllowanceFactory.build ─▶ CompanyAllowance entity (domain)
  │   2. build PayrollSetting value object            (domain)
  │   3. entity.update_attributes → validate!         (domain rules)
  │   4. transaction { Repository.update ; RecalcService }  (infrastructure → MySQL)
  ▼
  controller renders JSON 200   (or 400 if a domain validation failed)
```

---

## 6. How to read an unknown module file: the 4-question method

When you open any file under `app/modules/`:

1. **Which domain?** → first part of the namespace (`Payroll`, `Attendance`...).
2. **Which layer?** → `Application` / `Domain` / `Infrastructure` in the namespace or path.
3. **What kind?** → `Services` / `Models` / `Repositories` / `Factories` / `QueryServices`.
4. **What does its `execute` / `initialize` take and return?** → read the `sig`.

Example: `Payroll::Infrastructure::QueryServices::FetchEmployeePayrollStatementTemplatesQueryService` → Payroll / infrastructure / query service / takes `company_id:, employee_ids:`, returns an array of hashes.

---

## 7. Practice (15 min)

**Practice 1: layer check.** For each, say which layer it belongs to and whether it may use ActiveRecord:
a) `app/modules/payroll/domain/services/check_collection_target_of_child_support_fund_service.rb`
b) `app/modules/payroll/infrastructure/factories/company_allowance_factory.rb`
c) `app/modules/payroll/application/services/update_company_allowance_service.rb`

**Practice 2: boundary judge.** Allowed or banned by `Dependency/OverBoundary`?
a) `Attendance::Foo` calls `EmployeeMaster::Bar`
b) `EmployeeMaster::Foo` calls `Payroll::Bar`
c) `CommentSerializer` calls `CommentRepository`
d) `Payroll::Foo` calls `Attendance::Bar`

**Practice 3: explore.** Run `ls ~/Josh_Files/freee-payroll/app/modules/` and list the domains. Pick one you don't know (e.g. `hr_portal`) and find its `domain/models` folder. Pick one model and do the 4-question method.

**Practice 4: entity or value object?** a) An employee b) A postal address c) A pay period "2026-09" d) A contract document #88

*(Answers: 1 a) domain, no AR; b) infrastructure, yes; c) application. It orchestrates; in practice it opens transactions and calls repositories. Direct AR use is discouraged (see the `@deprecated` note on passing AR objects). 2 a) allowed b) banned c) banned d) allowed. 4 a) entity b) value object c) value object d) entity.)*

---

## 8. Summary

- `app/modules/<domain>/` = code organized **by business domain**, then by **layer**.
- **application** (use cases) → **domain** (models + pure rules) ← **infrastructure** (repositories, factories, query services; the only layer touching ActiveRecord).
- **Entity** = identity matters; **value object** = values matter, frozen. Unsure? Pick value object.
- Domains form a **DAG**: `Payroll → Attendance → EmployeeMaster`; reverse references are banned by the `Dependency/OverBoundary` cop, and serializers can't call repositories.
- Controllers/serializers stay **outside** `app/modules`.
- Domain validation errors are turned into HTTP 400 **at the API boundary** (controller `rescue`).

➡️ Now take **Test18.md**.
