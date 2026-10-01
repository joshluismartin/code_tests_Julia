# Test 19 — Jobs, External Services, and Operations Helpers

**Rules:** Don't look at Lesson19.md. Then check `Answers19.md`.
**Total:** 40 points

---

## Part A — Multiple choice (1 point each)

**1.** *Asynchronous* work means:
- a) Work done now while the user waits
- b) Work put aside to run later in the background
- c) Work done by the browser
- d) Work that never finishes

**2.** Where must a **new** background job be written?
- a) `app/resque_jobs/`
- b) `lib/tasks/`
- c) `app/jobs/` as an ActiveJob
- d) `app/services/`

**3.** Which method enqueues an ActiveJob to run later?
- a) `perform`
- b) `perform_later`
- c) `enqueue!`
- d) `run_async`

**4.** Why pass `company_id:` instead of a `Company` object to a job?
- a) Ruby can't pass objects
- b) Jobs are serialized and stored in a queue (Redis); simple values like ids are safe to store, then the record is loaded inside `perform`
- c) It's faster to type
- d) Sorbet requires it

**5.** What backend actually runs freee-payroll's ActiveJobs in production?
- a) Sidekiq
- b) Resque
- c) Cron
- d) AWS Lambda

**6.** *gRPC* is:
- a) A frontend library
- b) A system for calling functions on another service over the network, using protobuf messages
- c) A database
- d) A test framework

**7.** Which call gets the registered gRPC client for the Shift service?
- a) `Proto::Shift.new`
- b) `Omega::GRPC.client(:shift)`
- c) `ShiftClient.connect`
- d) `Rails.grpc(:shift)`

**8.** What does `ReachableChecker.execute(message: '...')` do in **production**?
- a) Crashes the request
- b) Reports to Bugsnag and continues
- c) Nothing at all
- d) Writes to `Rails.logger`

**9.** ... and in **test**?
- a) Raises an error
- b) Reports and continues
- c) Nothing
- d) Skips the test

**10.** Which should you use to encrypt a sensitive column?
- a) Rails' `encrypts`
- b) `DatabaseColumnEncryptor`
- c) `Integrations::Encryptable`
- d) `PasswordHasher`

**11.** Passwords should be:
- a) Encrypted with `DatabaseColumnEncryptor`
- b) Stored as plain text
- c) Hashed with `PasswordHasher`
- d) Base64 encoded

---

## Part B — Fill in the blanks (1 point each)

**12.** Every ActiveJob in freee-payroll inherits from ______.

**13.** The line that chooses which queue a job goes into is `______ :exlong`.

**14.** Instead of `Rails.logger`, report problems with `Omega::______.notify`.

**15.** `DatabaseColumnEncryptor.encrypt(value)` returns a hash with two keys: `:______` and `:______`.

**16.** Heavy read-only queries should be wrapped in `DatabaseSelector.______ do ... end`.

**17.** Oneshot tasks don't set the company in `Omega::______`, so calling another microservice from a task can fail with `AccessViolationError`.

---

## Part C — Fill in the missing code (2 points each)

**18.** Complete this modern job:

```ruby
class ExportEmployeesCsvJob < ______
  ______ :short

  def ______(company_id:, bulk_process_id:, **_metadata)
    company = Company.find(company_id)
    EmployeeMaster::ExportEmployeesCsvService.execute(company_id: company.id, bulk_process_id: bulk_process_id)
  end
end
```

**19.** Enqueue that job from a service (one line):

```ruby
ExportEmployeesCsvJob.______(company_id: company.id, bulk_process_id: bulk_process.id)
```

**20.** Complete this gRPC call and error report:

```ruby
begin
  Omega::GRPC.______(:shift).delete_employee_data(
    ::Proto::Shift::DeleteEmployeeDataRequest.______(employee_id: employee.id),
  )
rescue => e
  Omega::Bugcap.______(e)
end
```

---

## Part D — Write code (4 points each)

**21.** Convert this legacy Resque job into a modern ActiveJob (same queue, keyword args), and write how it would be enqueued:

```ruby
# app/resque_jobs/recalc_bonus_job.rb
class RecalcBonusJob
  extend WorkerModule

  @queue = :long

  def self.perform(company_id, bonus_id)
    company = Company.find(company_id)
    Payroll::RecalcBonusService.execute(company: company, bonus_id: bonus_id)
  end
end
# enqueued with: Resque.enqueue(RecalcBonusJob, company.id, bonus.id)
```

**22.** Write a model method `save_note!(plaintext)` (illustrative) that encrypts `plaintext` with `DatabaseColumnEncryptor` into the columns `encrypted_note` and `encrypted_note_iv`, sets both to `nil` if the text is blank, then calls `save!`.

---

## Part E — Answer in a sentence or two (2 points each)

**23.** Why should a job's `perform` method be "thin" and just call a service?

**24.** When would you use `Omega::Bugcap.notify` instead of `ReachableChecker`?

**25.** In `company_info_repository.rb`, gRPC errors from nest-auth are caught and re-raised as `HrErrors::UserError`. Why do this in the repository?

---

## Part F — Read real code (3 points)

**26.** From `app/services/attendance/daily_attendance_tags/imports/enqueue_service.rb`:

```ruby
      if is_importing?(company, daily_attendance_tag_import)
        raise ::HrErrors::UserError.new('勤怠タグインポート中です。インポート終了後に再度お試しください。')
      end

      ActiveRecord::Base.transaction do
        daily_attendance_tag_import.enqueued!
        ImportDailyAttendanceTagsJob.perform_later(
          company_id: company.id,
          daily_attendance_tag_import_id: daily_attendance_tag_import.id,
        )
      end

      { id: T.must(daily_attendance_tag_import.id), status: daily_attendance_tag_import.status }
```

a) What happens if an import for the same period is already running?
b) What does `enqueued!` do?
c) When this method returns, has the import actually finished? What will the frontend probably do next?
