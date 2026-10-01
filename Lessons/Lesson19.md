# Lesson 19 — Jobs, External Services, and Operations Helpers

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Understand background jobs (ActiveJob vs legacy Resque), how freee-payroll talks to other freee microservices (gRPC), and the "operations" helpers you'll see everywhere: `Omega::Bugcap.notify`, `ReachableChecker`, `DatabaseSelector`, `DatabaseColumnEncryptor`.
**You need:** Lessons 17–18. Open `~/Josh_Files/freee-payroll/`.

---

## 1. Synchronous vs asynchronous

| Term | Meaning in plain words |
|---|---|
| **Synchronous** | Do the work *now*, while the user waits. A normal HTTP request is synchronous. |
| **Asynchronous (async)** | Say "I'll do it later", answer the user right away, and do the work in the background. |
| **Background job** | A unit of work that runs later, outside the HTTP request. |
| **Queue** | A waiting line of jobs. Jobs are added at the back and taken from the front. |
| **Enqueue** | Put a job into a queue. |
| **Worker** | A separate process that takes jobs from a queue and runs them. |
| **Redis** | A fast in-memory data store. Resque keeps its queues in Redis. |
| **Resque** | The job system (gem) freee-payroll uses under the hood. |
| **ActiveJob** | Rails' standard way to write jobs. It sits on top of a backend like Resque. |
| **Backend / adapter** | The actual system that stores and runs jobs. Here: `config/environments/production.rb:66` → `config.active_job.queue_adapter = :resque`. |

Why use jobs? Some tasks are slow: importing a CSV of 5,000 employees, calculating payroll for a whole company, exporting survey results. If the HTTP request did all that, the browser would time out. So:

```
Browser ── POST /import ──▶ Controller ──▶ Service
                                              │ 1. mark import "enqueued"
                                              │ 2. Job.perform_later(ids)  ──▶  [ Redis queue :exlong ]
◀── 200 { status: "enqueued" } ───────────────┘                                     │
                                                                          Worker process picks it up
(browser polls "is it done yet?")                                                  │
                                                                     Job#perform → CreateService.execute
```

---

## 2. ActiveJob in freee-payroll

### 2.1 Rule: new jobs go in `app/jobs/` as ActiveJob

`app/README.md:18`: *"New async jobs are written with ActiveJob in `app/jobs/`. Adding to `app/resque_jobs/` is forbidden (checked by `spec/job_class_policies_spec.rb`)."*

`spec/job_class_policies_spec.rb:5-18`:

```ruby
  it 'forbit adding new class into app/resque_jobs, please add into app/jobs as ActiveJob' do
    # TODO: @C-FO/hr_quality 将来的に存在しないようにする。これは既存classで落ちないようにするための暫定対応なので新規で追加しないこと。
    ignore_list = [
      AttendanceImportJob,
      Attendances::SpecialHolidayPoolsExportJob,
      CalcStatementJob,
      KintaiPlusEmployeeSyncJob,
    ]

    resque_job_classes =
      (Dir.glob('**/*.rb', base: "#{Rails.application.root}/app/resque_jobs")).sort.map do |file|
        file.delete_suffix('.rb').classify.constantize
      end
    expect(resque_job_classes).to contain_exactly(*ignore_list)
  end
```

Only those four legacy Resque jobs are allowed to exist. There's even a team skill for migrating them: `.claude/skills/convert-to-active-job/SKILL.md` ("WorkerModuleベースのJobをActiveJobベースに変換する").

### 2.2 A real job

`app/jobs/import_daily_attendance_tags_job.rb:1-14`:

```ruby
# typed: true
# 日次勤怠タグインポートのjob
class ImportDailyAttendanceTagsJob < ApplicationJob
  queue_as :exlong

  def perform(company_id:, daily_attendance_tag_import_id:, **_metadata)
    company = Company.find(company_id)
    ::Attendance::DailyAttendanceTags::Imports::CreateService.execute(
      company: company,
      daily_attendance_tag_import_id: daily_attendance_tag_import_id,
      parser: ::Attendance::DailyAttendanceTags::Imports::Parser.new,
    )
  end
end
```

| Part | Meaning |
|---|---|
| `< ApplicationJob` | All jobs inherit from `ApplicationJob` (which inherits `ActiveJob::Base`) |
| `queue_as :exlong` | Which queue this job goes in. Queues in `app/jobs`: `:exlong` (34 jobs), `:short` (12), `:long` (7), plus a couple of special ones. Names hint at how long jobs run. |
| `def perform(...)` | The method the worker calls later. |
| `company_id:` (an **id**, not a `Company` object) | Jobs are **serialized** (turned into text) and stored in Redis. Pass simple values like ids; load the record inside `perform`. |
| `**_metadata` | "Accept any other keyword args and ignore them." The leading `_` means "intentionally unused". |
| Body | A job should be **thin**: load, then call a **Service**. Business logic stays in services. |

### 2.3 Enqueuing a job: `perform_later`

`app/services/attendance/daily_attendance_tags/imports/enqueue_service.rb:17-25`:

```ruby
      ActiveRecord::Base.transaction do
        daily_attendance_tag_import.enqueued!
        ImportDailyAttendanceTagsJob.perform_later(
          company_id: company.id,
          daily_attendance_tag_import_id: daily_attendance_tag_import.id,
        )
      end

      { id: T.must(daily_attendance_tag_import.id), status: daily_attendance_tag_import.status }
```

- `perform_later(...)` = enqueue now, run later. (`perform_now` runs immediately; mostly used in tests.)
- `enqueued!` is an enum bang method: it sets status to `enqueued` and saves (Lesson 12).
- The service returns right away with `{ id:, status: 'enqueued' }`; the frontend polls for progress.

Another pair: `app/jobs/export_survey_result_job.rb` (`queue_as :short`) is enqueued at `app/modules/application/hr_survey/application/services/exports/enqueue_service.rb:36`:

```ruby
                ExportSurveyResultJob.perform_later(company_id: company.id, bulk_process_id: bulk_process.id)
```

### 2.4 What `ApplicationJob` adds

`app/jobs/application_job.rb:3-22`:

```ruby
class ApplicationJob < ActiveJob::Base
  attr_accessor :caller_context # 呼び出し元の文脈

  include OmegaRails::Job::ActivityLoggable
  include OmegaRails::Job::ContextHandler
  include OmegaRails::Job::InternalSessionHandler
  include OmegaRails::Job::FeatureFlagHandler

  # @see https://github.com/C-FO/omega-ruby#activejob
  before_enqueue :inject_omega_context_attrs
  before_enqueue :inject_omega_feature_flag_attrs
  before_enqueue :inject_omega_internal_session
  after_perform :clear_request_store # freee-payroll独自のhooks
  ...
  around_perform :run_with_omega_core_context
  ...
```

| Term | Meaning |
|---|---|
| **Omega** | freee's internal shared Ruby framework (`gem 'omega'`) for context, errors, gRPC, feature flags, auth. You'll see `Omega::...` everywhere. |
| **Context** (`Omega::Context`) | "Who is doing this, for which company, in which request?" information carried along with the work. |
| **Job callbacks** | Like model callbacks: `before_enqueue`, `before_perform`, `around_perform`, `after_perform`. |

The important idea: when a job is enqueued during a web request, `ApplicationJob` **saves the caller's context** (company, user, which employees they may access, feature flags) and **restores it** when the job runs later in a worker. Without that, the job wouldn't know whose permissions apply. That's also why `ApplicationJob#serialize` adds a `'CUSTOM_caller_context'` hash.

### 2.5 Legacy Resque jobs (recognize them)

`app/resque_jobs/calc_statement_job.rb:3-6`:

```ruby
class CalcStatementJob
  extend WorkerModule

  @queue = :payroll_calculator
```

It has `self.perform(company_id, eps_id, ...)` with **positional** args, and is enqueued like `Resque.enqueue_to(queue_name, CalcStatementJob, company_id, id, ...)` (`app/models/employee_payroll_statement.rb:1861`). The repo has ~98 `perform_later` calls but only 4 `Resque.enqueue*` calls, so ActiveJob is the norm.

| Modern (ActiveJob) | Legacy (Resque) |
|---|---|
| `app/jobs/` | `app/resque_jobs/` |
| `class X < ApplicationJob` | `class X` + `extend WorkerModule` |
| `queue_as :short` | `@queue = :short` |
| `def perform(company_id:, ...)` | `def self.perform(company_id, ...)` |
| `X.perform_later(company_id: 1)` | `Resque.enqueue(X, 1)` |

---

## 3. Talking to other freee services (microservices & gRPC)

### 3.1 Vocabulary

| Term | Meaning |
|---|---|
| **Microservice** | A separate server program owning one function (e.g. permissions, members, notifications). |
| **RPC** | *Remote Procedure Call*: calling a function that runs on another computer as if it were local. |
| **gRPC** | Google's RPC system; fast, typed. Most freee service-to-service calls use it. |
| **Protocol Buffers (protobuf, "proto")** | The language for defining gRPC messages and services. Ruby classes are generated from `.proto` files. |
| **`proto-*` gems** | Generated Ruby clients: `proto-shift`, `proto-nest_auth`, `proto-wolf`, `proto-sekisyo`, `proto-ohmu`, `proto-delibird`... (see `Gemfile`) |
| **Stub** | The client object that sends RPCs. |
| **Endpoint** | The network address of a service (from an environment variable like `SHIFT_GRPC_ENDPOINT`). |
| **Timeout** | Give up if the other service doesn't answer within N seconds. |

Nicknames you'll meet: **nest-auth** (members, users, company info), **Sekisyo** (permissions), **Wolf** (approval flows), **Ohmu** (organization/group memberships), **Delibird** (notifications), **Shift** (shift planning).

### 3.2 Registering stubs

`config/initializers/omega.rb:16` starts `Omega::GRPC.configure do |config|`. Production endpoints come from environment variables (lines 17-29), e.g. `shift_endpoint = ENV['SHIFT_GRPC_ENDPOINT']`. Then, line 85:

```ruby
  config.register_stub(:shift, Proto::Shift::ShiftService::Stub, shift_endpoint, with_internal_session: true)
```

= "When code asks for `:shift`, use this generated stub, pointing at this address, and attach an internal session (so the other service knows who's calling)." There are 48 `register_stub` calls in that file.

### 3.3 Making a call

Simplest real call, `app/models/employee.rb:700-706`:

```ruby
    begin
      Omega::GRPC.client(:shift).delete_employee_data(
        ::Proto::Shift::DeleteEmployeeDataRequest.new(employee_id: self.id),
      )
    rescue => e
      Omega::Bugcap.notify(e)
    end
```

1. `Omega::GRPC.client(:shift)`: get the registered stub.
2. `.delete_employee_data(...)`: the RPC method (defined in the proto).
3. `Proto::Shift::DeleteEmployeeDataRequest.new(employee_id: ...)`: the request message.
4. If it fails, **report the error but don't crash** (deleting an employee here shouldn't fail because Shift is down).

Reading a response, `app/models/concerns/company/ohmu_usable.rb:10-21`:

```ruby
  def has_any_group_memberships?(date:)
    return false unless company_ohmu_sync_status.synced?
    response =
      ::Omega::GRPC.client(:ohmu_group_membership_v2).list_group_memberships(
        ::Ohmu::V2::ListGroupMembershipsRequest.new(
          company_id: id,
          date: ::Ohmu::Converters::GoogleDate.from_ruby_date(date),
          pagination: ::Ohmu::V2::CursorPagination.new(limit: 1),
        ),
      )
    response.group_memberships.any?
  end
```

### 3.4 Turning remote errors into our errors

`app/repositories/application/company_info_repository.rb:47-68`:

```ruby
    Omega::GRPC.client(:nest_org_info).update_company_info(
      Proto::NestAuth::UpdateCompanyInfoRequest.new(**req_attrs),
      timeout_sec: 5.0,
    )
  rescue Omega::Errors::BaseError => e
    case e.error_code
    when 'NAT-0054-0001', 'NAT-0054-0002'
      hr_error = HrErrors::UserError.new(e.user_message.presence || e.message)
      e.invalid_fields&.each { |f| hr_error.add_field_error(name: f.name, message: f.user_message) }
      raise hr_error
    when 'NAT-0054-0003'
      raise HrErrors::UserError.new('対象の事業所が見つかりませんでした。', status: :not_found)
    else
      raise HrErrors::UserError.new(
              '事業所情報の更新に失敗しました',
              status: :internal_server_error,
              notify_error: true,
              severity: :warning,
              original_exception: e,
            )
    end
  end
```

- `timeout_sec: 5.0`: don't wait forever.
- The other service returns **error codes**; the **repository** translates them into `HrErrors::UserError` with a proper HTTP status. That's why remote calls live in the **infrastructure / repository** layer: callers never see gRPC details.

### 3.5 HTTP clients in `lib/`

Some services are called over plain HTTP (REST) instead of gRPC. Their wrappers live in `lib/*_client.rb`: `delibird_client.rb`, `freee_sign_client.rb`, `egov_client.rb`, `mynaportal_client.rb`, `salesforce_client.rb`, `google_maps_client.rb`, `freee_postal_code_client.rb`, and more. Pattern: `class XClient; class << self; ...` with methods that send requests and return data.

### 3.6 A trap: tasks with no company context

`docs/knowledge/general/coding-standards/task-microservice-call-check.md`, summarized:

- Calls to other services (nest-auth, Delibird...) automatically send **`Omega::Context.current`'s `company_id`** as metadata ("on whose authority am I calling?"), *separately* from the `company_id` in the request body.
- The receiving service compares them. If the context has no company, you get **`AccessViolationError`**.
- Web requests and jobs set the context automatically. **Oneshot/permanent tasks (`Omega::Task::Base`) do not**, so code that works in a batch can fail when copied into a task.
- It's sneaky: local DB access still works, and the gRPC call is often buried inside a service (e.g. a notification).

---

## 4. Operations helpers

"Operations" = keeping the system healthy in production: noticing errors, being safe with data, handling load.

### 4.1 `Omega::Bugcap.notify`: report an error

| Term | Meaning |
|---|---|
| **Bugsnag** | The error-monitoring service. When an error is reported, engineers see it on a dashboard with details. |
| **Bugcap** | freee's Omega wrapper that sends errors to Bugsnag. |
| **Severity** | How serious: `:error`, `:warning`, `:info`. |

Rule (`app/README.md:15`): **don't use `Rails.logger`** (disabled in production, cop `Hr/UsedRailsLogger`). Use `Omega::Bugcap.notify`.

`app/services/attendance/shift_work_schedules/imports/get_importing_status_service.rb:101-120`:

```ruby
      # 設計上到達しうる分岐で通知をしたいので、ReachableChecker ではなく Bugcap.notify を使用する。
      Omega::Bugcap.notify(
        StandardError.new('進行中のまま閾値を超えたシフトのまとめて保存を failed へ終端した'),
        :warning,
        custom: {
          bulk_process_id: bulk_process.id,
          company_id: bulk_process.company_id,
          ...
        },
      )
      'failed'
    rescue StandardError => e
      # 書き込み・通知の失敗で status 応答を巻き添えにしない。
      Omega::Bugcap.notify(e)
      'failed'
    end
```

- Arguments: the error, the severity, and `custom:` extra info shown in Bugsnag's CUSTOM tab (ids help debugging).
- First comment: *"We want to be notified on a branch that CAN be reached by design, so use Bugcap.notify, not ReachableChecker."*

You don't usually call `notify` for normal errors in controllers: `ApplicationController#handle_error` already reports unexpected errors automatically (Lesson 15).

### 4.2 `ReachableChecker`: "this line should never run"

`lib/reachable_checker.rb:12-15`:

```ruby
    def execute(company_id: nil, employee_id: nil, message: nil, custom: {})
      raise UnexpectedReachedError.new(message)
    rescue UnexpectedReachedError => error
      raise if Rails.env.development? || Rails.env.test?
```

- **development/test:** it **raises** → your test/dev server crashes loudly, so you find the bug.
- **staging/production:** it **reports** to Bugsnag via `Omega::Bugcap.notify(error, :warning, ...)` and **continues**, so users aren't broken.

Real use, `app/controllers/api/private/employee_yearend_adjustments2014_controller.rb:11-14`:

```ruby
  before_action do
    T.bind(self, Api::Private::EmployeeYearendAdjustments2014Controller)
    ::ReachableChecker.execute(message: 'このエンドポイントは利用されていない想定です') if current_company.readonly_obsolete_nencho_enabled?
  end
```

"This endpoint is assumed to be unused." If Bugsnag stays silent for a while, the team can safely delete it. `docs/knowledge/general/coding-standards/reachable-checker.md` also shows using it to **compare old vs new logic** during refactoring (notify if results differ).

| Use... | When the branch... |
|---|---|
| `ReachableChecker.execute(message:)` | **should never** be reached (a bug or dead code if it is) |
| `Omega::Bugcap.notify(e, :warning, custom:)` | **can** be reached by design, but you want to know |
| `raise HrErrors::UserError` | is a normal user mistake → show a message |

### 4.3 `DatabaseSelector.connected_to_reader`: use the read replica

Recall Lesson 13: a **read replica** is a read-only copy of the database. Heavy reads there take load off the primary.

`app/presenters/generate_income_tax_fixed_amount_tax_reductions_table_presenter.rb:57-62`:

```ruby
    ::DatabaseSelector.connected_to_reader do
      @records =
        Payroll::Infrastructure::QueryServices::IncomeTaxFixedAmountTaxReductionQueryService.execute(
          company_id: company.id,
        )
    end
```

Rules (`docs/knowledge/general/coding-standards/read-replica-usage.md`): don't call `connected_to(role: :reading)` directly; go through `DatabaseSelector` (which notifies and retries on the primary if a write sneaks in). **Don't use it** right after your own write (replication lag) or inside a transaction.

### 4.4 `DatabaseColumnEncryptor`: encrypt sensitive columns

| Term | Meaning |
|---|---|
| **Encryption** | Scrambling data with a **key** so only someone with the key can read it. |
| **Decryption** | Unscrambling it back. |
| **IV (initialization vector)** | Random extra input so encrypting the same text twice gives different results. Stored next to the encrypted value. |
| **KMS** | AWS Key Management Service; holds the master key that protects our data key. |
| **Hashing** | One-way scrambling (can't be reversed). Used for passwords. |

Rules (`app/README.md:28`, `docs/knowledge/general/coding-standards/database-encryption.md`):
- Encrypt with **`DatabaseColumnEncryptor`** (`lib/database_column_encryptor.rb`), **not** Rails' built-in `encrypts`. It meets freee's encryption standard (two key layers: AES-256-GCM data key, itself encrypted by KMS).
- **Passwords: hash, don't encrypt**, using `PasswordHasher` (`lib/password_hasher.rb`).
- `Integrations::Encryptable` is an older one-layer scheme; don't use it for new code.
- What to encrypt is decided by freee's security-level definitions.

Real usage, `app/models/employee_hr_portal_sub_action.rb:28-35`:

```ruby
    if @note_plaintext.blank?
      self.encrypted_note = nil
      self.encrypted_note_iv = nil
    else
      encrypted = DatabaseColumnEncryptor.encrypt(@note_plaintext)
      self.encrypted_note = encrypted[:encrypted]
      self.encrypted_note_iv = encrypted[:iv]
    end
```

`encrypt(value)` returns `{ iv:, encrypted: }` (both base64 strings), saved into a **column pair** `encrypted_<name>` + `encrypted_<name>_iv` (convention described in the class comment, `lib/database_column_encryptor.rb:10-15`). `decrypt(iv, encrypted)` reverses it.

---

## 5. Practice (15 min)

**Practice 1: modern or legacy?** Open `app/jobs/export_survey_result_job.rb` and `app/resque_jobs/calc_statement_job.rb`. List three differences.

**Practice 2: find a job's caller.** Pick any job in `app/jobs/` and search for `JobName.perform_later` in the repo. What service/controller enqueues it? What queue is it on?

**Practice 3: which helper?** For each situation choose `ReachableChecker`, `Omega::Bugcap.notify`, `HrErrors::UserError`, or `DatabaseSelector.connected_to_reader`:
a) An old endpoint you believe nobody calls anymore
b) The user uploaded a CSV with no rows
c) A heavy read-only report query
d) A retry loop that *sometimes* gives up by design, and you want a record of it

**Practice 4: gRPC reading.** In `config/initializers/omega.rb`, find the `register_stub` for `:ohmu_group_membership_v2`. Which environment variable gives its endpoint in production?

*(Answers: 1: `app/jobs` vs `app/resque_jobs`; `< ApplicationJob` vs `extend WorkerModule`; `queue_as :short` vs `@queue =`; keyword `perform` instance method vs positional `self.perform`. 3 a) ReachableChecker b) HrErrors::UserError c) DatabaseSelector.connected_to_reader d) Omega::Bugcap.notify. 4: `ohmu_endpoint = ENV['OHMU_API_SERVER']`.)*

---

## 6. Summary

- **Background jobs** do slow work later. New jobs: `app/jobs/`, `class X < ApplicationJob`, `queue_as`, `def perform(company_id:, ...)`, enqueue with `X.perform_later(...)`. Pass **ids**, keep jobs **thin** (call a service).
- `app/resque_jobs/` is legacy (4 allowed classes); the backend is still Resque.
- `ApplicationJob` carries the caller's **Omega context** (company, user, permissions, flags) into the job.
- Other freee services are called via **gRPC** (`Omega::GRPC.client(:name).rpc_method(Proto::...Request.new(...))`) or HTTP clients in `lib/*_client.rb`; translate remote errors into `HrErrors` in the repository.
- Tasks don't set company context → microservice calls can fail with `AccessViolationError`.
- `Omega::Bugcap.notify` (report, don't use `Rails.logger`), `ReachableChecker` (should-never-happen), `DatabaseSelector.connected_to_reader` (read replica), `DatabaseColumnEncryptor` (encryption; passwords → `PasswordHasher`).

➡️ Now take **Test19.md**.
