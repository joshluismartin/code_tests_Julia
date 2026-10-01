# Test 15 — Private API, OpenAPI, and Error Handling

**Rules:** Don't look at Lesson15.md. Write your answers under each question, then check `Answers15.md`.
**Total:** 40 points

---

## Part A — Multiple choice (1 point each)

**1.** What is an OpenAPI (Swagger) file?
- a) A Ruby class
- b) A YAML description of an API's paths, parameters, and request/response shapes
- c) A database migration
- d) A CSS file

**2.** "Schema-driven development" means:
- a) Write the database schema last
- b) Write the API schema first and treat it as the single source of truth
- c) Never write a schema
- d) Generate the schema from the controller

**3.** Where does a Private API controller's schema file go?
- a) `docs/api/`
- b) Next to the controller, named `<controller名>.swagger.yaml`
- c) In `front/`
- d) In `config/routes.rb`

**4.** Which of these should you **NOT** put in a Private API controller's swagger file?
- a) The path
- b) Request parameters
- c) The success response
- d) Error response definitions

**5.** What does the `committee` gem do?
- a) Formats Ruby code
- b) Validates real requests/responses against the OpenAPI schema at runtime
- c) Sends emails
- d) Runs database migrations

**6.** For `/api/p/`, a response that doesn't match the schema in **production**:
- a) Crashes the request with a 500
- b) Is reported (Bugsnag) but the request keeps working
- c) Is silently ignored with no report
- d) Deletes the schema

**7.** The Domain API schema is written in:
- a) JSON
- b) TypeSpec (`.tsp`)
- c) Ruby
- d) CSV

**8.** `HrErrors::UserError` without a `status:` argument returns which HTTP status?
- a) 200
- b) 400
- c) 404
- d) 500

**9.** Which error is sent to error monitoring (Bugsnag) **by default**?
- a) `HrErrors::UserError`
- b) `HrErrors::SystemError`
- c) Both
- d) Neither

**10.** A *middleware* is:
- a) A database
- b) Code between the web server and the controller that sees every request/response
- c) A React component
- d) A test helper

---

## Part B — Fill in the blanks (1 point each)

**11.** The command that combines all small swagger files into one root file is `make ______`.

**12.** The command that regenerates the TypeScript client is `make ______`.

**13.** The generated TS function for `GET /api/p/employee_display_names/{id}` is named `______`.

**14.** Errors from an `ErrorHandleable` controller are rendered as JSON with the three keys `______`, `______`, `______`.

**15.** An `ActiveRecord::RecordInvalid` error is converted by `ErrorHandleable` into an `HrErrors::______`.

**16.** To reload the Private API schema without restarting Rails in development, set the environment variable `______=1`.

---

## Part C — Fill in the missing code (2 points each)

**17.** Complete the schema for a required integer path parameter `id`:

```yaml
      parameters:
        - name: id
          in: ______
          required: ______
          schema:
            type: ______
```

**18.** Raise an error telling the user the file wasn't found, with HTTP 404:

```ruby
raise ::HrErrors::______.new('ファイルが見つかりません。', ______)
```

**19.** Render a list of comments with `CommentSerializer`, without a root key:

```ruby
ActiveModel::ArraySerializer.new(
  comments,
  ______: CommentSerializer,
  authors_by_user_id: authors_by_user_id,
  root: ______,
)
```

---

## Part D — Write the code (4 points each)

**20.** Write the full swagger YAML (illustrative) for `GET /api/p/company_holiday_count` that returns `{ "count": integer }` (required). Include `openapi`, `info`, `paths`.

**21.** Write the controller (illustrative) `Api::Private::CompanyHolidayCountsController` with a `show` action that calls `::Attendance::Infrastructure::Repositories::HolidayRepository.find_list_by_company_id(company_id: ...)` and returns `{ count: <number of results> }`. Follow all freee-payroll conventions. Then list the **other files** you'd need to create or change for this endpoint.

---

## Part E — Answer in a sentence or two (2 points each)

**22.** Why is it useful that the TypeScript client is generated from the same schema committee validates against?

**23.** Why do most freee-payroll controllers not contain `rescue` blocks?

**24.** In `MemberRepository.find_one_by_employee_id`, why is "employee not found" a `UserError` but "employee exists but member missing" a `SystemError`?

---

## Part F — Read real code (4 points)

**25.** Given this real code:

```ruby
# app/controllers/api/private/employee_display_names_controller.rb
def show
  member =
    ::HrMaster::Infrastructure::Repositories::MemberRepository.find_one_by_employee_id(
      company_id: current_company.id,
      employee_id: show_params[:id].to_i,
    )

  render json: { id: member.employee_id, display_name: member.display_name }
end
```

and `raise HrErrors::UserError.new('従業員が見つかりませんでした') if employee.nil?` in the repository:

- a) A request `GET /api/p/employee_display_names/99999` is made for an ID that doesn't exist in the company. What HTTP status and JSON `message` does the client receive?
- b) Why is `.to_i` needed?
- c) Why is `company_id: current_company.id` passed?
- d) If a developer changed the render to `{ id: member.employee_id, name: member.display_name }`, what would happen in development, and why?
