# Test 9 — Rails Overview: MVC and the Request Lifecycle

**Rules:** Don't look at Lesson9.md. Write your answers under each question, then check `Answers9.md`.
**Total:** 35 points

---

## Part A — Multiple choice (1 point each)

**1.** In MVC, which part represents data and the rules about that data?
- a) View
- b) Controller
- c) Model
- d) Middleware

**2.** In MVC, which part receives the request and decides what to send back?
- a) Model
- b) Controller
- c) View
- d) Migration

**3.** "Convention over configuration" means:
- a) You must configure every connection by hand
- b) If you follow Rails' naming rules, Rails connects things automatically
- c) Conventions are only for the frontend
- d) Configuration files are forbidden

**4.** What is Zeitwerk?
- a) A database
- b) Rails' autoloader, which requires file paths to match class names
- c) A testing library
- d) A JSON format

**5.** Which environment is used while automated tests run?
- a) development
- b) staging
- c) test
- d) production

**6.** What is middleware?
- a) Code every request passes through before reaching the controller
- b) The middle column of a database table
- c) A React component
- d) A type of migration

**7.** In freee-payroll, what do most backend controllers return?
- a) Full HTML pages built with ERB
- b) JSON (for React to display)
- c) PDF files only
- d) CSV files only

**8.** What does `bin/rails c` do?
- a) Compiles the app
- b) Opens an interactive Ruby console with the Rails app loaded
- c) Creates a controller
- d) Clears the cache

---

## Part B — Fill in the blanks (1 point per blank)

**9.** The model class `Comment` is stored by convention in the table named ______ and in the file `app/models/______.rb`.

**10.** A public method in a controller that handles a route is called an ______.

**11.** The request values from the URL, query string, and body are available in a controller through ______.

**12.** freee-payroll sets the time zone to ______ and the default language to `:______` (see `config/application.rb`).

**13.** Code in `app/modules/` uses a ______ architecture; the rest of `app/` is called ______.

---

## Part C — Zeitwerk: file ⇄ class (1 point each)

Write the class name:

**14.** `app/models/attendance/check_status.rb` → ______
**15.** `app/services/payroll/salaries/unpublish_service.rb` → ______
**16.** `app/modules/hr_master/domain/models/member.rb` → ______

Write the file path:

**17.** `Api::Private::CompanyDailyProrateSettingsController` → ______
**18.** `Web::HolidaysController` → ______

---

## Part D — Order the steps (3 points)

**19.** Put these steps of a request in the correct order (write the letters):

- A. Repository/model queries MySQL
- B. Router matches the URL in `config/routes.rb`
- C. Response JSON is sent back to the browser
- D. Middleware (e.g. committee) runs
- E. Controller action runs (after its before_actions)

---

## Part E — Answer in a sentence or two (2 points each)

**20.** Why is `app/views/web/app.html.erb` almost empty (only `<div id="app"></div>`)?

**21.** What does it tell you when you see `class_name:` or `foreign_key:` on an association?

**22.** Name the four environments in `config/environments/` and say in one phrase what each is for.

---

## Part F — Read real code (5 points)

`app/controllers/api/private/employee_display_names_controller.rb`:

```ruby
class Api::Private::EmployeeDisplayNamesController < Api::Private::BaseController
  include ErrorHandleable

  freee_doorkeeper_for :show

  def show
    member =
      ::HrMaster::Infrastructure::Repositories::MemberRepository.find_one_by_employee_id(
        company_id: current_company.id,
        employee_id: show_params[:id].to_i,
      )

    render json: { id: member.employee_id, display_name: member.display_name }
  end

  private

  def show_params
    params.permit(:id)
  end
end
```

**23.** (1) Which URL + HTTP method reaches this `show` action? (The route is `resources :employee_display_names, only: [:show]` inside `/api/p`.)
**24.** (1) In which file is `MemberRepository` defined?
**25.** (1) Why is `current_company.id` passed to the repository?
**26.** (1) What is `.to_i` for on `show_params[:id]`?
**27.** (1) What does the response body look like for employee 1 named 給与太郎?
