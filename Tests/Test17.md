# Test 17 — Rails-Extended Layers: Services, Repositories, Values, Presenters

**Rules:** Don't look at Lesson17.md. Write answers under each question. Then check `Answers17.md`.
**Total:** 38 points

---

## Part A — Multiple choice (1 point each)

**1.** What is *business logic*?
- a) Code that draws buttons
- b) The rules of the business, e.g. how overtime is paid
- c) The database schema
- d) The CI configuration

**2.** In freee-payroll, a Service's only public method must be:
- a) An instance method `call`
- b) A class method `run`
- c) A class method `execute`
- d) Any method, as long as it has a `sig`

**3.** Which file enforces the Service rule?
- a) `.rubocop.yml`
- b) `spec/service_class_policies_spec.rb`
- c) `app/README.md`
- d) `config/application.rb`

**4.** What is a *value object*?
- a) An object that is equal to another if they have the same values, and usually can't be changed
- b) An ActiveRecord model with an `id`
- c) Any object that holds a number
- d) A JSON response

**5.** What does `ActsAsValueObject#initialize` do after `super`?
- a) Saves to the database
- b) Calls `validate!` then `freeze`
- c) Calls `to_json`
- d) Nothing

**6.** Which layer should normally **open the transaction** when several tables are written?
- a) The repository
- b) The serializer
- c) The service (the caller)
- d) The model's `after_save`

**7.** A class that turns an object into a Hash for a JSON response is a:
- a) Presenter
- b) Serializer
- c) Decorator
- d) Repository

**8.** What should you use instead of `Rails.logger` to report problems?
- a) `puts`
- b) `binding.b`
- c) `Omega::Bugcap.notify`
- d) `raise`

**9.** Why do services and repositories have `ignore_list`s in their policy specs?
- a) To list the classes that are the best examples
- b) To let old (legacy) classes that break the rule keep passing, until they're fixed
- c) To skip slow tests
- d) To list classes that should be deleted immediately

**10.** *Coupling* means:
- a) How much one piece of code depends on another
- b) Two tests running together
- c) A database join
- d) Using two gems

---

## Part B — Fill in the blanks (1 point each)

**11.** In a repository method name, `_by_` means filtered by a ______, and `_at` means as of a point in ______.

**12.** A method defined inside `class << self` is a ______ method.

**13.** In a serializer, the thing being serialized is available as ______, and extra data passed in by the controller is in ______.

**14.** New value objects are written as a `T::______` that includes `ActsAsValueObject`.

**15.** A direct query like `Employee.where(...)` outside allowed places is caught by the cop `Freee/______`; use a model ______ or a Repository instead.

---

## Part C — Fill in the missing code (2 points each)

**16.** Make this a correct modern service (fill the blanks):

```ruby
class Attendance::ArchiveHolidayService
  ______ ______ ______
    extend T::Sig

    sig { params(company_id: Integer, holiday_id: Integer).void }
    def ______(company_id:, holiday_id:)
      ActiveRecord::Base.______ do
        ...
      end
    end
  end
end
```

**17.** Rename each method so it **passes** `spec/repository_class_policies_spec.rb` (write the new name):
- a) `def find_employee(company_id:, employee_id:)` → returns one employee by id
- b) `def find_all_retired(company_id:)` → returns many employees filtered by "retired"
- c) `def find_list(company_id:, date:)` → many records as of a date

**18.** This repository method fails the policy for a different reason. Fix the signature:

```ruby
class Payroll::Infrastructure::Repositories::BonusRepository
  class << self
    def find_one_by_id(company_id, bonus_id)
      ...
    end
  end
end
```

---

## Part D — Write code (4 points each)

**19.** Write a value object `Payroll::Domain::Models::PayMonth` with:
- two constant fields `year` (Integer) and `month` (Integer)
- a validation that `month` is in `1..12`
- it must be validated on creation and immutable (use the freee-payroll way)

Then write one line creating a valid one and one line that would raise an error.

**20.** Write a modern service `Payroll::UpdateRoundTypeService` whose `execute(company_id:, round_type:)`:
- has a Sorbet `sig` (returns `void`)
- inside a transaction, calls `Payroll::Infrastructure::Repositories::RoundTypeRepository.save(company_id: company_id, round_type: round_type)`
- has one private helper method (anything you like) placed correctly so the policy spec still passes

---

## Part E — Answer in a sentence or two (2 points each)

**21.** Why does freee-payroll want repositories to hide *how* data is fetched? Give one concrete benefit.

**22.** `Api::Private::CompanyAllowancesController` does not include `ErrorHandleable`, but CI passes. Why? Should your new controller skip it too?

**23.** Explain the difference between a Presenter and a Decorator.

---

## Part F — Read real code (3 points)

**24.** From `app/services/employee_master/mass_updates/bulk_processes/delete_bulk_process_service.rb`:

```ruby
class EmployeeMaster::MassUpdates::BulkProcesses::DeleteBulkProcessService
  class << self
    extend T::Sig

    def execute(company:, bulk_process_id:)
      ActiveRecord::Base.transaction do
        company
          .bulk_processes
          .where(type: BulkProcess.deletable_history_types_for_employee_bulk_updates)
          .where(status: %i[initialized tmp_uploaded failed undone cancel_requested canceled])
          .find(bulk_process_id)
          .destroy_logically!
      end
    end
  end
end
```

a) How do you call this service? (write the call)
b) What happens if the bulk process has status `complete`?
c) Does `destroy_logically!` physically remove the row? What does it do?
